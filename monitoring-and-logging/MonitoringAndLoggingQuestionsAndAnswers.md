# Monitoring & Logging Interview Questions and Answers

A focused collection of monitoring and logging interview questions for DevOps engineers — covering Datadog, CloudWatch, Prometheus, Grafana, OpenSearch, Fluentbit, Loki, Promtail, and real-world migration scenarios.

---

## Table of Contents

**Monitoring:**
- [Monitoring Fundamentals](#monitoring-fundamentals)
- [Datadog](#datadog)
- [AWS CloudWatch](#aws-cloudwatch)
- [Prometheus & Grafana](#prometheus--grafana)

**Logging:**
- [Logging Fundamentals](#logging-fundamentals)
- [Fluentbit & Log Pipelines](#fluentbit--log-pipelines)
- [OpenSearch (ELK Alternative)](#opensearch-elk-alternative)
- [Grafana Loki & Promtail](#grafana-loki--promtail)
- [Migration Scenarios](#migration-scenarios)

---

# MONITORING

---

## Monitoring Fundamentals

### 1. What is monitoring in DevOps and why is it important?

**Answer:** Monitoring is the practice of collecting, analyzing, and alerting on metrics, logs, and traces from infrastructure and applications to ensure system health, performance, and availability.

**The three pillars of observability:**

| Pillar | What it captures | Example tools |
|--------|-----------------|---------------|
| **Metrics** | Numeric measurements over time (CPU, memory, latency) | Prometheus, Datadog, CloudWatch |
| **Logs** | Timestamped event records | Loki, OpenSearch, CloudWatch Logs |
| **Traces** | Request flow across services (distributed tracing) | Jaeger, Datadog APM, X-Ray |

**Key monitoring concepts:**
- **Golden signals:** Latency, traffic, errors, saturation
- **RED method (services):** Rate, Errors, Duration
- **USE method (infrastructure):** Utilization, Saturation, Errors
- **SLI/SLO/SLA:** Service Level Indicator → Objective → Agreement

---

## Datadog

### 2. What is Datadog and what are its key features?

**Answer:** Datadog is a **SaaS-based observability platform** that provides unified monitoring, logging, APM, and security across infrastructure, applications, and services.

**Key features:**
- **Infrastructure monitoring** — Metrics from hosts, containers, cloud services
- **APM (Application Performance Monitoring)** — Distributed tracing, latency analysis
- **Log Management** — Centralized log collection, search, and analytics
- **Synthetics** — Proactive endpoint monitoring (HTTP, browser, API tests)
- **RUM (Real User Monitoring)** — Frontend performance tracking
- **Dashboards & Alerts** — Custom dashboards, anomaly detection, PagerDuty/Slack integration
- **Integrations** — 700+ integrations (AWS, Kubernetes, Docker, Terraform, etc.)

**Architecture:**
```
Application/Host → Datadog Agent → Datadog Backend (SaaS)
                                        ↓
                                  Dashboards / Alerts / APM
```

---

### 3. How does the Datadog Agent work?

**Answer:** The Datadog Agent is a lightweight process that runs on hosts to collect metrics, logs, and traces and forwards them to the Datadog platform.

```bash
# Install Datadog Agent (Linux)
DD_API_KEY=<YOUR_API_KEY> DD_SITE="datadoghq.com" \
  bash -c "$(curl -L https://install.datadoghq.com/scripts/install_script_agent7.sh)"

# Agent configuration
cat /etc/datadog-agent/datadog.yaml
# api_key: <KEY>
# site: datadoghq.com
# logs_enabled: true
# apm_config:
#   enabled: true

# Agent status
sudo datadog-agent status

# Check integrations
sudo datadog-agent configcheck
```

**Agent components:**
- **Core Agent** — Collects system metrics (CPU, memory, disk, network)
- **APM Agent** — Receives traces from instrumented applications (port 8126)
- **Log Agent** — Tails log files and forwards to Datadog
- **Process Agent** — Collects live process and container data
- **Cluster Agent** (Kubernetes) — Central point for cluster-level monitoring

---

### 4. How do you set up monitoring and alerting in Datadog?

**Answer:**

**Monitors (alerts):**

| Monitor Type | Use Case |
|-------------|----------|
| **Metric** | CPU > 90% for 5 min |
| **Log** | Error count > 100 in 10 min |
| **APM** | P95 latency > 500ms |
| **Synthetics** | API endpoint returns non-200 |
| **Composite** | Multiple conditions combined |
| **Anomaly** | Deviation from baseline |
| **Forecast** | Predict disk full in 48 hours |

```python
# Create monitor via API
{
    "type": "metric alert",
    "query": "avg(last_5m):avg:system.cpu.user{env:production} by {host} > 90",
    "name": "High CPU Usage on {{host.name}}",
    "message": "CPU usage is above 90% on {{host.name}}. @slack-ops-alerts @pagerduty-oncall",
    "tags": ["env:production", "team:platform"],
    "options": {
        "thresholds": {"critical": 90, "warning": 80},
        "notify_no_data": true,
        "no_data_timeframe": 10,
        "renotify_interval": 30
    }
}
```

**Dashboards:**
- **Screenboards** — Free-form layout (status pages, executive views)
- **Timeboards** — Time-synced widgets (debugging, correlation)
- **Template variables** — `$env`, `$service`, `$host` for dynamic filtering

---

### 5. How does Datadog integrate with Kubernetes?

**Answer:**

```yaml
# Datadog Agent DaemonSet (Helm)
helm repo add datadog https://helm.datadoghq.com
helm install datadog-agent datadog/datadog \
  --set datadog.apiKey=<API_KEY> \
  --set datadog.logs.enabled=true \
  --set datadog.apm.portEnabled=true \
  --set datadog.processAgent.enabled=true \
  --set clusterAgent.enabled=true

# Pod annotations for log collection
metadata:
  annotations:
    ad.datadoghq.com/myapp.logs: |
      [{
        "source": "java",
        "service": "myapp",
        "log_processing_rules": [{
          "type": "multi_line",
          "name": "java_stacktrace",
          "pattern": "\\d{4}-\\d{2}-\\d{2}"
        }]
      }]

# Custom metrics via DogStatsD
# Application sends: statsd.increment('myapp.request.count', tags=['env:prod'])
```

**Cluster Agent benefits:**
- Reduces API server load (single point for cluster metadata)
- Horizontal Pod Autoscaler based on Datadog metrics
- Cluster checks (e.g., checking external endpoints from one agent)

---

## AWS CloudWatch

### 6. What is AWS CloudWatch and what are its components?

**Answer:** CloudWatch is AWS's native monitoring and observability service.

| Component | Purpose |
|-----------|---------|
| **Metrics** | Numeric data from AWS services (EC2, RDS, Lambda, etc.) |
| **Logs** | Log collection and analysis (Log Groups, Log Streams) |
| **Alarms** | Trigger actions based on metric thresholds |
| **Dashboards** | Custom visual dashboards |
| **Events / EventBridge** | React to state changes (EC2 stopped, deployment complete) |
| **Insights** | Log Insights (query), Container Insights, Lambda Insights |
| **Synthetics** | Canary scripts for endpoint monitoring |

```bash
# Custom metric
aws cloudwatch put-metric-data \
  --namespace "MyApp" \
  --metric-name "OrderCount" \
  --value 42 \
  --dimensions Environment=Production,Service=OrderService

# Create alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "HighCPU" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 90 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:123456:ops-alerts \
  --dimensions Name=InstanceId,Value=i-0abc123
```

---

### 7. How do you query logs with CloudWatch Log Insights?

**Answer:**

```sql
-- Top 10 error messages in last 1 hour
fields @timestamp, @message
| filter @message like /ERROR/
| stats count(*) as error_count by @message
| sort error_count desc
| limit 10

-- P95 latency by API endpoint
filter @type = "REPORT"
| stats avg(@duration) as avg_ms,
        pct(@duration, 95) as p95_ms,
        max(@duration) as max_ms
  by @requestId

-- Find Lambda cold starts
filter @type = "REPORT"
| filter @initDuration > 0
| stats count(*) as cold_starts,
        avg(@initDuration) as avg_init_ms
  by bin(30m)

-- EC2 instance errors
fields @timestamp, @message
| filter @logStream like /i-0abc123/
| filter @message like /(?i)error|exception|fatal/
| sort @timestamp desc
| limit 50
```

---

### 8. What is the difference between Datadog and CloudWatch?

**Answer:**

| Feature | Datadog | CloudWatch |
|---------|---------|-----------|
| **Type** | SaaS (multi-cloud) | AWS-native |
| **Multi-cloud** | Yes (AWS, GCP, Azure, on-prem) | AWS only |
| **APM/Tracing** | Built-in, powerful | X-Ray (separate service) |
| **Log analytics** | Rich search, patterns, pipelines | Log Insights (SQL-like) |
| **Dashboards** | Advanced, drag-and-drop | Basic widgets |
| **Anomaly detection** | ML-based, built-in | Limited (anomaly detection band) |
| **Integrations** | 700+ out of the box | AWS services primarily |
| **Cost** | Per host/log volume (expensive at scale) | Pay-per-use (cheap for basic) |
| **Kubernetes** | First-class (Cluster Agent, live containers) | Container Insights (basic) |
| **Best for** | Multi-cloud, complex apps, full observability | AWS-only, cost-sensitive, basic monitoring |

---

## Prometheus & Grafana

### 9. What is Prometheus and how does it work?

**Answer:** Prometheus is an **open-source metrics monitoring and alerting system**, a CNCF graduated project. It uses a **pull model** — scraping metrics from targets at regular intervals.

**Architecture:**
```
Targets (apps, exporters) ← Prometheus Server (scrape, store) → Alertmanager → Slack/PD
                                      ↓
                                   Grafana (visualization)
```

**Key concepts:**

| Concept | Description |
|---------|-------------|
| **Metric** | Time-series data identified by name + labels |
| **Label** | Key-value pair for dimensional filtering |
| **Scrape** | Pulling metrics from `/metrics` endpoint |
| **Exporter** | Translates service metrics to Prometheus format |
| **PromQL** | Query language for metrics |
| **Alertmanager** | Routes and manages alert notifications |

**Metric types:**

| Type | Use | Example |
|------|-----|---------|
| **Counter** | Only increases (resets on restart) | `http_requests_total` |
| **Gauge** | Goes up and down | `temperature_celsius` |
| **Histogram** | Distribution of values in buckets | `request_duration_seconds` |
| **Summary** | Similar to histogram, calculates quantiles client-side | `rpc_duration_seconds` |

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
```

---

### 10. What is PromQL and how do you write useful queries?

**Answer:** PromQL (Prometheus Query Language) is used to query and aggregate time-series data.

```promql
# Instant vector — current CPU usage per instance
node_cpu_seconds_total{mode="idle"}

# Rate — requests per second over last 5 minutes
rate(http_requests_total[5m])

# Error rate percentage
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m])) * 100

# P95 latency from histogram
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# Memory usage percentage
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Top 5 pods by CPU usage
topk(5, sum(rate(container_cpu_usage_seconds_total{namespace="production"}[5m])) by (pod))

# Disk will be full in 24 hours (prediction)
predict_linear(node_filesystem_free_bytes{mountpoint="/"}[6h], 24*3600) < 0

# Alert rule — high error rate
- alert: HighErrorRate
  expr: sum(rate(http_requests_total{status=~"5.."}[5m])) by (service) > 0.05
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "High error rate on {{ $labels.service }}"
```

---

### 11. How does Prometheus Alertmanager work?

**Answer:** Alertmanager handles alerts fired by Prometheus — it deduplicates, groups, routes, and sends notifications.

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/xxx/yyy/zzz'

route:
  receiver: 'default'
  group_by: ['alertname', 'namespace']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
    - match:
        severity: warning
      receiver: 'slack-warnings'

receivers:
  - name: 'default'
    slack_configs:
      - channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: '<PD_KEY>'

  - name: 'slack-warnings'
    slack_configs:
      - channel: '#warnings'

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'namespace']
```

**Key features:**
- **Grouping** — Combines related alerts into one notification
- **Inhibition** — Suppresses lower-severity alerts when critical is firing
- **Silencing** — Temporarily mute alerts (e.g., during maintenance)
- **Routing** — Route to different receivers by labels

---

### 12. What is Grafana and how does it integrate with Prometheus?

**Answer:** Grafana is an open-source **visualization and dashboarding platform** that supports multiple data sources.

**Common data sources:** Prometheus, Loki, CloudWatch, Elasticsearch/OpenSearch, InfluxDB, MySQL, PostgreSQL

**Key features:**
- Panels (time series, gauge, stat, bar, table, heatmap, logs)
- Template variables for dynamic dashboards
- Alerting (unified alerting across data sources)
- Annotations (mark events on graphs)
- Provisioning (dashboards-as-code via JSON/YAML)

```yaml
# Grafana datasource provisioning (Prometheus)
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    access: proxy
    isDefault: true

  - name: Loki
    type: loki
    url: http://loki:3100
    access: proxy
```

**Useful Prometheus dashboard panels:**
```promql
# CPU usage per node
100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100

# Pod restarts
sum(increase(kube_pod_container_status_restarts_total[1h])) by (namespace, pod)
```

---

### 13. What are Prometheus exporters and which ones are commonly used?

**Answer:** Exporters are agents that translate third-party metrics into Prometheus format, exposing them on a `/metrics` endpoint.

| Exporter | What it monitors |
|----------|-----------------|
| **Node Exporter** | Linux host metrics (CPU, memory, disk, network) |
| **kube-state-metrics** | Kubernetes object states (pods, deployments, nodes) |
| **cAdvisor** | Container resource usage (built into kubelet) |
| **Blackbox Exporter** | Probe endpoints (HTTP, DNS, TCP, ICMP) |
| **MySQL Exporter** | MySQL server metrics |
| **MongoDB Exporter** | MongoDB server metrics |
| **Redis Exporter** | Redis server metrics |
| **Nginx Exporter** | Nginx stub_status metrics |
| **CloudWatch Exporter** | AWS CloudWatch metrics into Prometheus |

```yaml
# Blackbox exporter — probe HTTP endpoints
- job_name: 'blackbox-http'
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets:
        - https://api.example.com/health
        - https://web.example.com
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: blackbox-exporter:9115
```

---

### 14. How does Prometheus compare to Datadog for monitoring?

**Answer:**

| Feature | Prometheus + Grafana | Datadog |
|---------|---------------------|---------|
| **Type** | Open-source, self-hosted | SaaS, managed |
| **Cost** | Free (infra cost only) | Per-host + per-metric pricing |
| **Setup** | Manual (Helm, operators) | Agent install + auto-discovery |
| **Retention** | Limited (use Thanos/Cortex for long-term) | 15 months default |
| **HA** | Requires Thanos/Cortex/Mimir | Built-in |
| **Multi-cluster** | Federation or Thanos | Native |
| **Query language** | PromQL (powerful, steep learning curve) | Point-and-click + custom queries |
| **Alerting** | Alertmanager (config-as-code) | UI-based + Terraform |
| **APM** | Separate (Jaeger/Tempo) | Built-in |
| **Logging** | Separate (Loki) | Built-in |
| **Best for** | Cost-sensitive, K8s-native, full control | Enterprise, multi-cloud, all-in-one |

**When to use Prometheus:** Kubernetes-native environments, cost-sensitive teams, need full control, open-source preference.

**When to use Datadog:** Enterprise teams, need all-in-one platform, multi-cloud, want managed service with low operational overhead.

---

# LOGGING

---

## Logging Fundamentals

### 15. What is centralized logging and why is it essential for DevOps?

**Answer:** Centralized logging aggregates logs from all services, hosts, and containers into a single searchable platform.

**Why it's essential:**
- **Microservices** generate logs across dozens/hundreds of services
- **Containers** are ephemeral — logs are lost when pods restart
- **Correlation** — Trace a request across multiple services
- **Alerting** — Trigger alerts on error patterns
- **Compliance** — Audit trail, retention requirements

**Typical logging pipeline:**
```
Application → Log Shipper → Message Queue (optional) → Log Backend → Dashboard
              (Fluentbit)    (Kafka)                    (OpenSearch    (Grafana/
                                                         or Loki)      Kibana)
```

**Log levels:**

| Level | When to use |
|-------|------------|
| `DEBUG` | Detailed diagnostic (development only) |
| `INFO` | Normal operations (request served, job started) |
| `WARN` | Recoverable issue (retry succeeded, degraded) |
| `ERROR` | Failure that needs attention (request failed, exception) |
| `FATAL/CRITICAL` | System unusable (startup failure, OOM) |

---

## Fluentbit & Log Pipelines

### 16. What is Fluent Bit and how does it differ from Fluentd?

**Answer:** Both are CNCF log processors, but with different profiles:

| Feature | Fluent Bit | Fluentd |
|---------|-----------|---------|
| **Written in** | C | Ruby + C |
| **Memory footprint** | ~450KB | ~40MB |
| **Performance** | Higher throughput, lower latency | Good, but heavier |
| **Plugin ecosystem** | ~100 plugins | ~1000 plugins |
| **Use case** | Edge, containers, Kubernetes DaemonSet | Aggregation layer, complex routing |
| **Configuration** | INI-like or YAML | Ruby-based config |

**Common architecture:**
```
Pods → Fluent Bit (DaemonSet, per node) → Fluentd (aggregator, optional) → Backend
```

**Fluent Bit Kubernetes DaemonSet config:**
```yaml
# fluent-bit.conf
[SERVICE]
    Flush         5
    Daemon        Off
    Log_Level     info
    Parsers_File  parsers.conf

[INPUT]
    Name              tail
    Path              /var/log/containers/*.log
    Parser            cri
    Tag               kube.*
    Refresh_Interval  5
    Mem_Buf_Limit     10MB
    Skip_Long_Lines   On

[FILTER]
    Name                kubernetes
    Match               kube.*
    Kube_URL            https://kubernetes.default.svc:443
    Kube_Tag_Prefix     kube.var.log.containers.
    Merge_Log           On
    Keep_Log            Off
    K8S-Logging.Parser  On
    K8S-Logging.Exclude On

[OUTPUT]
    Name            opensearch
    Match           *
    Host            opensearch.logging.svc
    Port            9200
    Index           kubernetes-logs
    Type            _doc
    Suppress_Type_Name On
    tls             On
    tls.verify      Off
    HTTP_User       admin
    HTTP_Passwd     ${OPENSEARCH_PASSWORD}

[OUTPUT]
    Name            loki
    Match           *
    Host            loki.logging.svc
    Port            3100
    Labels          job=fluentbit,namespace=$kubernetes['namespace_name'],pod=$kubernetes['pod_name']
    Auto_Kubernetes_Labels On
```

---

### 17. How do you build a robust log pipeline with Fluent Bit?

**Answer:**

**Key pipeline stages:**

| Stage | Purpose | Example |
|-------|---------|---------|
| **INPUT** | Collect logs | `tail`, `systemd`, `forward`, `tcp` |
| **PARSER** | Structure raw logs | JSON, regex, logfmt |
| **FILTER** | Enrich, transform, drop | `kubernetes`, `modify`, `grep`, `lua` |
| **OUTPUT** | Send to destinations | `opensearch`, `loki`, `s3`, `datadog`, `cloudwatch_logs` |

```yaml
# Filter — drop health check logs
[FILTER]
    Name    grep
    Match   kube.*
    Exclude log /healthz|/readyz|/livez/

# Filter — add custom fields
[FILTER]
    Name    modify
    Match   kube.*
    Add     environment production
    Add     cluster us-east-1

# Multi-output (fan out to OpenSearch + Loki + S3)
[OUTPUT]
    Name  opensearch
    Match *
    # ... config ...

[OUTPUT]
    Name  loki
    Match *
    # ... config ...

[OUTPUT]
    Name  s3
    Match *
    region        us-east-1
    bucket        logs-archive
    total_file_size 100M
    upload_timeout  10m
    s3_key_format /logs/%Y/%m/%d/$TAG/%H_%M_%S.gz
    compression   gzip
```

**Best practices:**
- Set `Mem_Buf_Limit` to prevent OOM on nodes
- Use `storage.type filesystem` for buffering during backend outages
- Filter out noisy logs (health checks, debug) before sending
- Use `Retry_Limit` to prevent infinite retries
- Monitor Fluent Bit itself (`/api/v1/metrics` endpoint)

---

## OpenSearch (ELK Alternative)

### 18. What is OpenSearch and how is it used for logging?

**Answer:** OpenSearch is an **open-source search and analytics engine** (AWS fork of Elasticsearch 7.10). It's used with OpenSearch Dashboards (fork of Kibana) for log analytics.

**Architecture:**
```
Fluent Bit → OpenSearch (indexing + search) → OpenSearch Dashboards (UI)
```

**Key concepts:**

| Concept | Description |
|---------|-------------|
| **Index** | Collection of documents (like a database table) |
| **Document** | Single log entry (JSON) |
| **Shard** | Horizontal partition of an index |
| **Replica** | Copy of a shard for HA |
| **Index pattern** | `kubernetes-logs-*` — matches multiple indices |
| **ISM (Index State Management)** | Lifecycle policy (hot → warm → cold → delete) |

```json
// ISM policy — auto-manage index lifecycle
{
  "policy": {
    "policy_id": "log-lifecycle",
    "default_state": "hot",
    "states": [
      {
        "name": "hot",
        "actions": [{ "rollover": { "min_size": "30gb", "min_index_age": "1d" } }],
        "transitions": [{ "state_name": "warm", "conditions": { "min_index_age": "7d" } }]
      },
      {
        "name": "warm",
        "actions": [{ "replica_count": { "number_of_replicas": 1 } }],
        "transitions": [{ "state_name": "delete", "conditions": { "min_index_age": "30d" } }]
      },
      {
        "name": "delete",
        "actions": [{ "delete": {} }]
      }
    ]
  }
}
```

---

### 19. What are the operational challenges of running OpenSearch?

**Answer:**

| Challenge | Description | Mitigation |
|-----------|-------------|------------|
| **Resource-heavy** | JVM heap, high CPU for indexing | Right-size instances, separate hot/warm nodes |
| **Shard management** | Too many shards degrade performance | Target 20-40GB per shard, use ILM/ISM |
| **Index explosion** | One index per day per service = thousands | Use data streams, rollover policies |
| **Cluster scaling** | Adding/removing nodes requires rebalancing | Use dedicated master nodes (3 minimum) |
| **Cost** | Storage + compute for full-text indexing | Archive cold data to S3, reduce replicas |
| **Query performance** | Complex aggregations are slow on large datasets | Use index templates, optimize mappings |
| **Upgrades** | Rolling upgrades can be complex | Blue-green cluster strategy |

---

## Grafana Loki & Promtail

### 20. What is Grafana Loki and how does it differ from OpenSearch/Elasticsearch?

**Answer:** Loki is a **log aggregation system** designed to be cost-effective and easy to operate. Unlike OpenSearch/Elasticsearch, **Loki does not index the log content** — it only indexes metadata (labels).

| Feature | Loki | OpenSearch/Elasticsearch |
|---------|------|------------------------|
| **Indexing** | Labels only (like Prometheus) | Full-text indexing |
| **Storage** | Object storage (S3, GCS) — very cheap | Local SSDs — expensive |
| **Resource usage** | Low (no full-text index) | High (JVM, CPU for indexing) |
| **Query speed** | Fast for label-filtered queries; slower for grep | Fast for any text search |
| **Query language** | LogQL (similar to PromQL) | OpenSearch DSL / SQL |
| **Cost at scale** | 10-50x cheaper than Elasticsearch | Expensive at high volume |
| **Complexity** | Simple to operate | Complex cluster management |
| **Best for** | Kubernetes, Prometheus users, cost-sensitive | Full-text search, complex analytics |

**Key insight:** Loki trades query flexibility for massive cost and operational savings. Perfect when you know *which service* has the issue and need to see its logs.

---

### 21. How does the Loki + Promtail + Grafana stack work?

**Answer:**

```
Application Logs → Promtail (or Fluent Bit) → Loki → Grafana
                   (collects, labels)        (stores)  (queries, visualizes)
```

**Promtail** — Log collection agent (similar to Fluent Bit but Loki-native):
```yaml
# promtail-config.yaml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    pipeline_stages:
      - cri: {}
      - json:
          expressions:
            level: level
            msg: message
      - labels:
          level:
      - match:
          selector: '{level="debug"}'
          action: drop
    relabel_configs:
      - source_labels: ['__meta_kubernetes_namespace']
        target_label: namespace
      - source_labels: ['__meta_kubernetes_pod_name']
        target_label: pod
      - source_labels: ['__meta_kubernetes_pod_label_app']
        target_label: app
```

**LogQL queries:**
```logql
# All error logs from myapp in production
{namespace="production", app="myapp"} |= "ERROR"

# Parse JSON logs and filter
{app="api"} | json | status >= 500

# Count errors per service over time
sum(rate({namespace="production"} |= "ERROR" [5m])) by (app)

# Top 10 error messages
{app="myapp"} |= "ERROR" | json | line_format "{{.message}}" | topk(10, sum(count_over_time({app="myapp"} |= "ERROR" [1h])) by (message))

# Latency percentile from logs
{app="api"} | json | unwrap duration | quantile_over_time(0.95, [5m])
```

---

### 22. When should you use Promtail vs Fluent Bit with Loki?

**Answer:**

| Feature | Promtail | Fluent Bit |
|---------|----------|-----------|
| **Native to Loki** | Yes — built for Loki | Loki output plugin |
| **Label extraction** | Rich pipeline stages | Basic label support |
| **Multi-output** | Loki only | Loki + OpenSearch + S3 + CloudWatch + more |
| **Resource usage** | Light | Very light (~450KB) |
| **Service discovery** | Kubernetes, journal, file | Kubernetes, file, systemd |
| **Use case** | Loki-only setup | Multi-destination pipelines |

**Recommendation:**
- **Promtail** — If Loki is your only log backend
- **Fluent Bit** — If you need to send logs to multiple backends (Loki + OpenSearch + S3) or during migrations

---

## Migration Scenarios

### 23. How did you migrate from Datadog to OpenSearch for logging? What were the challenges?

**Answer:**

**Why migrate:** Cost reduction — Datadog log management pricing scales per GB ingested, which becomes expensive at high volume.

**Migration approach:**
1. **Deploy OpenSearch cluster** — 3 master + 3 data nodes (hot/warm architecture)
2. **Deploy Fluent Bit** as DaemonSet alongside existing Datadog Agent
3. **Dual-write phase** — Send logs to both Datadog and OpenSearch simultaneously
4. **Validate** — Compare log completeness, latency, search capabilities
5. **Build dashboards** — Recreate Datadog log views in OpenSearch Dashboards
6. **Migrate alerts** — Recreate log-based monitors as OpenSearch alerting rules
7. **Cut over** — Disable Datadog log forwarding, keep Datadog for metrics/APM
8. **Decommission** — Remove Datadog log config from agents

**Challenges:**

| Challenge | How we handled it |
|-----------|-------------------|
| **Log format differences** | Built Fluent Bit parsers to match Datadog's auto-parsing |
| **Dashboard parity** | Iteratively rebuilt key dashboards; some features don't map 1:1 |
| **Alert migration** | Manually recreated; OpenSearch alerting syntax differs significantly |
| **Team training** | Trained teams on OpenSearch Dashboards and query syntax |
| **Operational overhead** | Now managing OpenSearch cluster (patching, scaling, shards) |
| **Missing features** | No Datadog-style log patterns, Live Tail less responsive |
| **Index management** | Had to implement ISM policies for retention/rollover |
| **Cost tradeoff** | Saved on Datadog bill but added infra + engineering cost |

**Advantages gained:**
- **70-80% cost reduction** on log storage at scale
- **Data sovereignty** — logs stay in our infrastructure
- **No vendor lock-in** — open-source, portable
- **Custom retention** — different retention per index/team

---

### 24. How did you migrate from OpenSearch to Grafana Loki? What were the advantages and challenges?

**Answer:**

**Why migrate:** Simplify operations and reduce cost further — OpenSearch cluster management was heavy, and most log queries were label-based (service/namespace/pod), not full-text.

**Migration approach:**
1. **Deploy Loki** — Simple mode or microservices mode with S3 backend
2. **Update Fluent Bit** — Add Loki output alongside OpenSearch output
3. **Dual-write phase** — Both backends receive logs
4. **Build Grafana dashboards** — Use LogQL for log panels alongside existing Prometheus metrics
5. **Validate** — Ensure all critical log queries work in LogQL
6. **Cut over** — Remove OpenSearch output from Fluent Bit
7. **Decommission** OpenSearch cluster

**Advantages:**

| Advantage | Detail |
|-----------|--------|
| **Massive cost reduction** | Loki uses S3 (object storage) — 10-50x cheaper than OpenSearch SSDs |
| **Simpler operations** | No JVM tuning, no shard management, no cluster rebalancing |
| **Unified stack** | Metrics (Prometheus) + Logs (Loki) + Traces (Tempo) — all in Grafana |
| **Label-based** | Same label model as Prometheus — easy correlation |
| **Horizontal scaling** | Stateless components, scale read/write independently |
| **Faster deployment** | Helm chart, minimal configuration |

**Challenges:**

| Challenge | Detail |
|-----------|--------|
| **No full-text index** | Grep-style search on content is slower for large time ranges |
| **LogQL learning curve** | Teams used to Kibana/OpenSearch DSL had to learn LogQL |
| **Label cardinality** | High-cardinality labels (request IDs, user IDs) crash Loki — must be in log content, not labels |
| **Dashboard recreation** | All OpenSearch Dashboards had to be rebuilt in Grafana |
| **Alerting differences** | OpenSearch alerting → Grafana alerting migration |
| **Missing features** | No equivalent to OpenSearch aggregations on log content |

**Key lesson:** Loki is ideal when your query pattern is: "Show me logs from service X in namespace Y for the last hour." It struggles when the pattern is: "Find all logs containing a specific request ID across all services."

---

### 25. How do you decide between Datadog, OpenSearch, and Loki for logging?

**Answer:**

```
                    Need full-text search
                    on log content?
                   /                \
                 YES                 NO
                  |                   |
          Need managed SaaS?    Budget-sensitive?
         /            \          /           \
       YES             NO     YES             NO
        |               |       |              |
    Datadog      OpenSearch    Loki        Datadog
    Logs         (self-hosted) (+ Grafana) (convenience)
```

| Factor | Datadog | OpenSearch | Loki |
|--------|---------|-----------|------|
| **Cost** | $$$$$ | $$$ | $ |
| **Ops overhead** | None (SaaS) | High (cluster mgmt) | Low |
| **Full-text search** | Excellent | Excellent | Limited (grep) |
| **K8s native** | Good | Moderate | Excellent |
| **Correlation** | Metrics+logs+traces unified | Logs only (add Prometheus) | Logs+metrics in Grafana |
| **Retention** | Expensive | Moderate (ISM policies) | Cheap (S3) |
| **Best for** | Enterprise, all-in-one | Full-text analytics, compliance | K8s, cost-sensitive, Prometheus users |

---

### 26. How do you implement structured logging for better observability?

**Answer:** Structured logging outputs logs as JSON instead of plain text, making them parseable by any log backend.

```python
# Python — structured logging
import structlog
import logging

structlog.configure(
    processors=[
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.add_log_level,
        structlog.processors.JSONRenderer()
    ],
    wrapper_class=structlog.BoundLogger,
)

logger = structlog.get_logger()
logger.info("order_placed",
    order_id="ORD-12345",
    user_id="USR-789",
    amount=99.99,
    currency="USD",
    service="order-service"
)
# Output: {"event":"order_placed","order_id":"ORD-12345","user_id":"USR-789","amount":99.99,...,"timestamp":"2024-01-15T10:30:00Z","level":"info"}
```

```go
// Go — structured logging with slog
slog.Info("order_placed",
    "order_id", "ORD-12345",
    "user_id", "USR-789",
    "amount", 99.99,
)
```

**Best practices:**
- Use JSON format — parseable by all backends
- Include: `timestamp`, `level`, `service`, `trace_id`, `message`
- Avoid logging sensitive data (PII, passwords, tokens)
- Use consistent field names across services
- Add `trace_id` / `correlation_id` for distributed tracing

---

### 27. How do you handle log retention, archival, and compliance?

**Answer:**

| Tier | Storage | Retention | Query speed | Cost |
|------|---------|-----------|-------------|------|
| **Hot** | SSD / primary index | 1-7 days | Fast | $$$ |
| **Warm** | HDD / reduced replicas | 7-30 days | Moderate | $$ |
| **Cold** | S3 / Glacier | 30-365 days | Slow | $ |
| **Archive** | S3 Glacier Deep | 1-7 years | Very slow (hours) | ¢ |

```bash
# S3 lifecycle policy for log archival
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-logs-archive \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "LogRetention",
      "Status": "Enabled",
      "Transitions": [
        {"Days": 30, "StorageClass": "STANDARD_IA"},
        {"Days": 90, "StorageClass": "GLACIER"},
        {"Days": 365, "StorageClass": "DEEP_ARCHIVE"}
      ],
      "Expiration": {"Days": 2555}
    }]
  }'
```

**Compliance considerations:**
- **PCI DSS** — 1 year minimum retention for audit logs
- **HIPAA** — 6 years for access logs
- **GDPR** — Right to erasure vs. retention requirements
- **SOC 2** — Demonstrate log integrity and access control

---

### 28. How do you monitor and alert on logs?

**Answer:**

**Grafana Loki alerting (via Grafana unified alerting):**
```yaml
# Grafana alert rule — high error rate from logs
- alert: HighLogErrorRate
  expr: |
    sum(rate({namespace="production"} |= "ERROR" [5m])) by (app) > 10
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "High error rate in {{ $labels.app }}"
```

**OpenSearch alerting:**
```json
{
  "name": "High Error Count",
  "type": "monitor",
  "inputs": [{
    "search": {
      "indices": ["kubernetes-logs-*"],
      "query": {
        "bool": {
          "must": [
            { "match": { "level": "ERROR" } },
            { "range": { "@timestamp": { "gte": "now-5m" } } }
          ]
        }
      }
    }
  }],
  "triggers": [{
    "condition": { "script": { "source": "ctx.results[0].hits.total.value > 100" } },
    "actions": [{
      "name": "slack-alert",
      "destination_id": "slack-webhook-id",
      "message_template": { "source": "High error count: {{ctx.results[0].hits.total.value}} errors in 5 min" }
    }]
  }]
}
```

---

### 29. How do you set up monitoring for a Kubernetes cluster end-to-end?

**Answer:** A complete K8s observability stack:

```
┌─────────────────────────────────────────────────────┐
│                    Grafana                           │
│  (Dashboards: metrics + logs + traces)              │
└────┬──────────────┬──────────────┬──────────────────┘
     │              │              │
┌────▼────┐  ┌──────▼─────┐  ┌────▼────┐
│Prometheus│  │   Loki     │  │  Tempo  │
│(metrics) │  │  (logs)    │  │(traces) │
└────▲────┘  └──────▲─────┘  └────▲────┘
     │              │              │
┌────┴────┐  ┌──────┴─────┐  ┌────┴────────────┐
│Exporters│  │ Fluent Bit │  │ App + OTel SDK  │
│node-exp │  │ (DaemonSet)│  │ (instrumented)  │
│kube-sm  │  └────────────┘  └─────────────────┘
└─────────┘
```

**Helm install (kube-prometheus-stack):**
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prom prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set grafana.enabled=true \
  --set alertmanager.enabled=true

# Add Loki
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  --namespace logging --create-namespace \
  --set fluent-bit.enabled=true \
  --set promtail.enabled=false
```

---

### 30. What are best practices for monitoring and logging in production?

**Answer:**

**Monitoring:**
- Define **SLIs/SLOs** before building dashboards
- Alert on **symptoms**, not causes (high latency, not high CPU)
- Use **golden signals** — latency, traffic, errors, saturation
- **Reduce alert noise** — group, deduplicate, set appropriate thresholds
- **Runbooks** — Link every alert to a runbook with troubleshooting steps
- Dashboard hierarchy: **Overview → Service → Component → Debug**

**Logging:**
- Use **structured logging** (JSON) everywhere
- Include **trace IDs** for distributed tracing correlation
- Filter noisy logs (health checks, debug) **at the shipper level**
- Set **retention policies** per log tier (hot/warm/cold)
- Monitor your **log pipeline itself** (Fluent Bit metrics, Loki ingestion rate)
- Never log **secrets, tokens, PII** — use scrubbing filters

**General:**
- **Tag everything** — service, environment, team, version
- **Automate** — dashboards-as-code, alerts-as-code (Terraform, Jsonnet)
- **Test alerts** — Regularly verify alerts fire and reach the right team
- **Capacity plan** — Monitor log volume growth, set budget alerts
