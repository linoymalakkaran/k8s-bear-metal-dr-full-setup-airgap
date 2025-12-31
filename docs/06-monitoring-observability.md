# Monitoring and Observability Guide

## Table of Contents
1. [Overview](#overview)
2. [Monitoring Stack](#monitoring-stack)
3. [Prometheus Setup](#prometheus-setup)
4. [Grafana Dashboards](#grafana-dashboards)
5. [Logging with Loki](#logging-with-loki)
6. [Alerting](#alerting)
7. [Distributed Tracing](#distributed-tracing)

## Overview

### Monitoring Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                            │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              Application Pods                              │ │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐                   │ │
│  │  │ App-1   │  │ App-2   │  │ App-N   │                   │ │
│  │  │ :8080   │  │ :8080   │  │ :8080   │                   │ │
│  │  │/metrics │  │/metrics │  │/metrics │                   │ │
│  │  └────┬────┘  └────┬────┘  └────┬────┘                   │ │
│  └───────┼────────────┼────────────┼────────────────────────┘ │
│          │            │            │                            │
│  ┌───────▼────────────▼────────────▼────────────────────────┐ │
│  │            Prometheus (Metrics Collection)               │ │
│  │  - Scrapes /metrics endpoints                            │ │
│  │  - Stores time-series data                               │ │
│  │  - Retention: 15 days                                    │ │
│  └───────┬──────────────────────────────────────────────────┘ │
│          │                                                      │
│  ┌───────▼──────────────────────────────────────────────────┐ │
│  │                 Grafana (Visualization)                   │ │
│  │  - Dashboards for metrics                                │ │
│  │  - Query Prometheus data                                 │ │
│  │  - Alerting UI                                           │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │             Loki (Log Aggregation)                         │ │
│  │  - Collects logs from all pods                            │ │
│  │  - Indexes labels, not full text                          │ │
│  │  - Integrated with Grafana                                │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │         AlertManager (Alert Management)                    │ │
│  │  - Receives alerts from Prometheus                         │ │
│  │  - Routes to Slack, Email, PagerDuty                      │ │
│  │  - Deduplication and grouping                             │ │
│  └────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

## Monitoring Stack

### Components

```yaml
monitoring_stack:
  prometheus:
    purpose: "Metrics collection and storage"
    retention: "15 days"
    scrape_interval: "30s"
    
  grafana:
    purpose: "Visualization and dashboards"
    datasources:
      - Prometheus
      - Loki
      - Jaeger (optional)
  
  loki:
    purpose: "Log aggregation"
    retention: "7 days"
    
  promtail:
    purpose: "Log shipping agent"
    deployment: "DaemonSet on all nodes"
  
  alertmanager:
    purpose: "Alert routing and management"
    receivers:
      - Slack
      - Email
      - PagerDuty
  
  node_exporter:
    purpose: "Hardware and OS metrics"
    deployment: "DaemonSet on all nodes"
  
  kube_state_metrics:
    purpose: "Kubernetes object metrics"
    deployment: "Single deployment"
```

## Prometheus Setup

### Install kube-prometheus-stack

```bash
#!/bin/bash
# File: monitoring/prometheus/install-prometheus-stack.sh

# Install complete Prometheus stack using Helm

set -e

NAMESPACE="monitoring"
RELEASE_NAME="kube-prometheus-stack"

echo "Installing Prometheus stack..."

# Add Prometheus community Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create namespace
kubectl create namespace ${NAMESPACE} || true

# Install kube-prometheus-stack
helm install ${RELEASE_NAME} prometheus-community/kube-prometheus-stack \
  --namespace ${NAMESPACE} \
  --set prometheus.prometheusSpec.retention=15d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=longhorn \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi \
  --set grafana.adminPassword=SecureGrafanaPassword123! \
  --set grafana.persistence.enabled=true \
  --set grafana.persistence.storageClassName=longhorn \
  --set grafana.persistence.size=10Gi \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.storageClassName=longhorn \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.resources.requests.storage=10Gi

echo "Waiting for Prometheus stack to be ready..."
kubectl wait --for=condition=Ready pods --all -n ${NAMESPACE} --timeout=300s

echo "Prometheus stack installed successfully!"
echo ""
echo "Access Grafana:"
echo "  kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80"
echo "  http://localhost:3000"
echo "  Username: admin"
echo "  Password: SecureGrafanaPassword123!"
echo ""
echo "Access Prometheus:"
echo "  kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090"
echo "  http://localhost:9090"
```

### Prometheus Configuration

```yaml
# File: monitoring/prometheus/prometheus-config.yaml

# ServiceMonitor for custom application
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: sample-app-metrics
  namespace: production
  labels:
    app: sample-app
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app: sample-app
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
    scrapeTimeout: 10s

---
# PodMonitor example
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: sample-app-pods
  namespace: production
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app: sample-app
  podMetricsEndpoints:
  - port: metrics
    path: /metrics
    interval: 30s
```

### Custom Metrics Examples

```yaml
# File: monitoring/prometheus/recording-rules.yaml

# PrometheusRule for custom recording rules
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: custom-recording-rules
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
  - name: application_rules
    interval: 30s
    rules:
    # Request rate per service
    - record: job:http_requests:rate5m
      expr: sum(rate(http_requests_total[5m])) by (job, service)
    
    # Error rate
    - record: job:http_errors:rate5m
      expr: sum(rate(http_requests_total{status=~"5.."}[5m])) by (job, service)
    
    # P95 latency
    - record: job:http_request_duration:p95
      expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (job, le))
    
    # Availability (percentage of non-5xx responses)
    - record: job:http_availability:ratio
      expr: |
        (
          sum(rate(http_requests_total{status!~"5.."}[5m])) by (job)
          /
          sum(rate(http_requests_total[5m])) by (job)
        ) * 100
```

## Grafana Dashboards

### Import Pre-built Dashboards

```bash
#!/bin/bash
# File: monitoring/grafana/import-dashboards.sh

# Import common Grafana dashboards

GRAFANA_URL="http://localhost:3000"
GRAFANA_USER="admin"
GRAFANA_PASS="SecureGrafanaPassword123!"

# Popular dashboard IDs from grafana.com
DASHBOARDS=(
  "6417"   # Kubernetes Cluster Monitoring
  "8588"   # Kubernetes Deployment Statefulset Daemonset metrics
  "12114"  # Kubernetes Nodes
  "13770"  # Kubernetes Pods
  "15172"  # Node Exporter Full
  "7249"   # Kubernetes Cluster (Prometheus)
)

for dashboard_id in "${DASHBOARDS[@]}"; do
  echo "Importing dashboard ${dashboard_id}..."
  
  curl -X POST \
    -H "Content-Type: application/json" \
    -u "${GRAFANA_USER}:${GRAFANA_PASS}" \
    "${GRAFANA_URL}/api/dashboards/import" \
    -d "{\"dashboard\": {\"id\": ${dashboard_id}}, \"overwrite\": true}"
done

echo "Dashboards imported successfully!"
```

### Custom Dashboard Example

```json
{
  "dashboard": {
    "title": "Application Performance Dashboard",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (service)",
            "legendFormat": "{{service}}"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Error Rate",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) by (service)",
            "legendFormat": "{{service}}"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Response Time (P95)",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (service, le))",
            "legendFormat": "{{service}} p95"
          }
        ],
        "type": "graph"
      }
    ]
  }
}
```

## Logging with Loki

### Install Loki Stack

```bash
#!/bin/bash
# File: monitoring/loki/install-loki.sh

# Install Loki for log aggregation

set -e

NAMESPACE="monitoring"

echo "Installing Loki stack..."

helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Loki and Promtail
helm install loki grafana/loki-stack \
  --namespace ${NAMESPACE} \
  --set loki.persistence.enabled=true \
  --set loki.persistence.storageClassName=longhorn \
  --set loki.persistence.size=50Gi \
  --set promtail.enabled=true \
  --set grafana.enabled=false

echo "Loki installed successfully!"

# Configure Loki as Grafana datasource
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: monitoring
  labels:
    grafana_datasource: "1"
data:
  loki-datasource.yaml: |-
    apiVersion: 1
    datasources:
    - name: Loki
      type: loki
      access: proxy
      url: http://loki:3100
      isDefault: false
      editable: true
EOF

echo "Loki datasource configured in Grafana"
```

### Loki Query Examples

```promql
# File: monitoring/loki/query-examples.promql

# All logs from production namespace
{namespace="production"}

# Logs containing "error"
{namespace="production"} |= "error"

# Logs from specific app
{namespace="production", app="sample-app"}

# Error logs excluding specific patterns
{namespace="production"} |= "error" != "connection reset"

# Parse JSON logs
{namespace="production"} | json

# Count errors in last 5 minutes
count_over_time({namespace="production"} |= "error" [5m])

# Rate of log lines
rate({namespace="production"}[5m])

# Filter by log level
{namespace="production"} | json | level="ERROR"
```

## Alerting

### AlertManager Configuration

```yaml
# File: monitoring/alertmanager/alertmanager-config.yaml

apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-kube-prometheus-stack-alertmanager
  namespace: monitoring
type: Opaque
stringData:
  alertmanager.yaml: |
    global:
      resolve_timeout: 5m
      slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
    
    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 12h
      receiver: 'default'
      routes:
      # Critical alerts to PagerDuty
      - match:
          severity: critical
        receiver: pagerduty
        continue: true
      
      # High severity to Slack with high priority
      - match:
          severity: high
        receiver: slack-high
      
      # Warning to Slack
      - match:
          severity: warning
        receiver: slack-warning
    
    receivers:
    - name: 'default'
      slack_configs:
      - channel: '#alerts'
        title: 'Alert: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
    
    - name: 'slack-high'
      slack_configs:
      - channel: '#critical-alerts'
        title: '🔴 HIGH: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        send_resolved: true
    
    - name: 'slack-warning'
      slack_configs:
      - channel: '#alerts'
        title: '⚠️ WARNING: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
    
    - name: 'pagerduty'
      pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_SERVICE_KEY'
        description: '{{ .GroupLabels.alertname }}: {{ .CommonAnnotations.summary }}'
```

### Alert Rules

```yaml
# File: monitoring/alertmanager/alert-rules.yaml

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kubernetes-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
  # Node Alerts
  - name: node_alerts
    interval: 30s
    rules:
    - alert: NodeDown
      expr: up{job="node-exporter"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Node {{ $labels.instance }} is down"
        description: "Node {{ $labels.instance }} has been down for more than 5 minutes."
    
    - alert: NodeHighCPU
      expr: (100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)) > 80
      for: 15m
      labels:
        severity: warning
      annotations:
        summary: "High CPU usage on {{ $labels.instance }}"
        description: "CPU usage is above 80% for 15 minutes."
    
    - alert: NodeHighMemory
      expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
      for: 10m
      labels:
        severity: critical
      annotations:
        summary: "High memory usage on {{ $labels.instance }}"
        description: "Memory usage is above 90%."
    
    - alert: NodeDiskSpaceLow
      expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 15
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Low disk space on {{ $labels.instance }}"
        description: "Disk space is below 15% on {{ $labels.mountpoint }}."
  
  # Pod Alerts
  - name: pod_alerts
    interval: 30s
    rules:
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is crash looping"
        description: "Pod {{ $labels.pod }} in namespace {{ $labels.namespace }} is restarting frequently."
    
    - alert: PodNotReady
      expr: kube_pod_status_phase{phase!~"Running|Succeeded"} == 1
      for: 15m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} not ready"
        description: "Pod has been in non-ready state for 15 minutes."
  
  # Application Alerts
  - name: application_alerts
    interval: 30s
    rules:
    - alert: HighErrorRate
      expr: (sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))) * 100 > 5
      for: 10m
      labels:
        severity: critical
      annotations:
        summary: "High error rate detected"
        description: "Error rate is above 5% for 10 minutes."
    
    - alert: HighResponseTime
      expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)) > 1
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "High response time (P95 > 1s)"
        description: "95th percentile response time is above 1 second."
  
  # Kubernetes Component Alerts
  - name: kubernetes_alerts
    interval: 30s
    rules:
    - alert: KubernetesAPIDown
      expr: up{job="kubernetes-apiservers"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Kubernetes API server is down"
        description: "Kubernetes API server has been down for 5 minutes."
    
    - alert: etcdDown
      expr: up{job="etcd"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "etcd is down"
        description: "etcd has been down for 5 minutes. CRITICAL!"
```

## Distributed Tracing

### Install Jaeger (Optional)

```bash
#!/bin/bash
# File: monitoring/tracing/install-jaeger.sh

# Install Jaeger for distributed tracing

set -e

NAMESPACE="monitoring"

echo "Installing Jaeger..."

# Install Jaeger operator
kubectl create namespace observability || true
kubectl create -f https://github.com/jaegertracing/jaeger-operator/releases/download/v1.51.0/jaeger-operator.yaml -n observability

# Wait for operator
kubectl wait --for=condition=available --timeout=300s deployment/jaeger-operator -n observability

# Create Jaeger instance
cat <<EOF | kubectl apply -f -
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-production
  namespace: ${NAMESPACE}
spec:
  strategy: production
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: http://elasticsearch:9200
  ingress:
    enabled: true
    hosts:
      - jaeger.example.com
EOF

echo "Jaeger installed successfully!"
```

## Next Steps

1. **Setup Alerts**: Configure AlertManager with your notification channels
2. **Create Dashboards**: Build custom Grafana dashboards for your applications
3. **Enable Tracing**: Instrument applications for distributed tracing
4. **Regular Review**: Review metrics and alerts weekly
5. **Capacity Planning**: Use metrics for infrastructure planning
