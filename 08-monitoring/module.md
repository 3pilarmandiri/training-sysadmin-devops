# BAB 8 — Monitoring & Observability

## Tujuan Pembelajaran

Setelah menyelesaikan bab ini, peserta mampu:
- Memahami konsep observability: metrics, logs, dan traces
- Mengkonfigurasi Prometheus untuk scraping metrics
- Membuat dashboard Grafana yang informatif
- Mengimplementasikan alerting dengan Alertmanager
- Mengumpulkan dan menganalisis log terpusat

---

## 8.1 Observability vs Monitoring

```
MONITORING (tradisional)                OBSERVABILITY (modern)
────────────────────────                ─────────────────────────
"Apakah sistem UP?"                     "Mengapa sistem berperilaku begini?"

Cek: server up/down                     Memahami state internal dari output
Alert: CPU > 90%                        Metrics + Logs + Traces
                                        → Analisis root cause
```

### Tiga Pilar Observability

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   METRICS   │  │    LOGS     │  │   TRACES    │
│             │  │             │  │             │
│ Angka yang  │  │ Rekaman     │  │ Alur request│
│ diukur dari │  │ kejadian    │  │ lintas      │
│ waktu ke    │  │ berteks     │  │ service     │
│ waktu       │  │             │  │             │
│ (Prometheus)│  │ (ELK/Loki)  │  │ (Jaeger)    │
└─────────────┘  └─────────────┘  └─────────────┘
```

**Kapan digunakan:**
- **Metrics**: "Request rate naik 300% jam 14:00"
- **Logs**: "Error: database connection refused di /api/users"
- **Traces**: "Request ini memakan 2 detik karena query DB lambat"

---

## 8.2 Prometheus

### Arsitektur Prometheus

```
Target Applications          Prometheus Server
(expose /metrics endpoint)
                            ┌────────────────────────────┐
 ┌─────────┐                │                            │
 │ App 1   │←── scrape ─────│  Retrieval Engine          │
 │ :9090   │                │  (setiap 15 detik pull)    │
 └─────────┘                │                            │
                            │  TSDB                      │
 ┌─────────┐                │  (Time Series Database)    │
 │ App 2   │←── scrape ─────│                            │
 │ :8080   │                │  HTTP API                  │
 └─────────┘                │  /api/v1/query             │
                            └────────────┬───────────────┘
 ┌─────────┐                             │
 │ Node    │←── scrape ─────             │
 │ Exporter│                     ┌───────▼────────┐
 └─────────┘                     │   Grafana      │
                                 │  (visualisasi) │
                          ┌──────▼──────────────┐ └────────────────┘
                          │  Alertmanager       │
                          │  (kirim notif)      │
                          └─────────────────────┘
```

### Jenis Metrics Prometheus

| Tipe | Penjelasan | Contoh |
|---|---|---|
| **Counter** | Hanya naik, tidak turun | Total HTTP requests, error count |
| **Gauge** | Bisa naik/turun | CPU usage, memory, jumlah pod aktif |
| **Histogram** | Distribusi nilai dalam bucket | Request latency (< 100ms, < 500ms, dll) |
| **Summary** | Seperti histogram, tapi hitung quantile | Latency p50, p90, p99 |

### Format Metrics (OpenMetrics/Prometheus)

```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",endpoint="/api/users",status="200"} 1483
http_requests_total{method="GET",endpoint="/api/users",status="500"} 12
http_requests_total{method="POST",endpoint="/api/orders",status="201"} 847

# HELP process_memory_bytes Current memory usage in bytes
# TYPE process_memory_bytes gauge
process_memory_bytes 47185920

# HELP http_request_duration_seconds HTTP request duration
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.1"} 24054
http_request_duration_seconds_bucket{le="0.5"} 33444
http_request_duration_seconds_bucket{le="1.0"} 34440
http_request_duration_seconds_bucket{le="+Inf"} 34459
http_request_duration_seconds_sum 17827.5
http_request_duration_seconds_count 34459
```

### Konfigurasi Prometheus

```yaml
# prometheus.yml
global:
  scrape_interval: 15s        # Scrape setiap 15 detik
  evaluation_interval: 15s    # Evaluasi rules setiap 15 detik
  scrape_timeout: 10s

# Alertmanager
alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

# Rule files untuk alerting
rule_files:
  - "rules/*.yml"

# Scrape configurations
scrape_configs:
  # Prometheus scrape dirinya sendiri
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]

  # Scrape Node Exporter
  - job_name: node
    static_configs:
      - targets:
          - "node1:9100"
          - "node2:9100"
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance

  # Scrape aplikasi Flask
  - job_name: flask-app
    metrics_path: /metrics
    static_configs:
      - targets: ["flask-app:8080"]
    scrape_interval: 30s

  # Service Discovery untuk Kubernetes
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
```

---

## 8.3 PromQL — Query Language Prometheus

```promql
# Counter — total requests
http_requests_total

# Filter by label
http_requests_total{job="flask-app", status="200"}

# Rate (perubahan per detik selama 5 menit terakhir)
rate(http_requests_total[5m])

# Irate (instantaneous rate)
irate(http_requests_total[5m])

# Error rate (%)
rate(http_requests_total{status=~"5.."}[5m]) /
rate(http_requests_total[5m]) * 100

# Persentase CPU usage
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage (%)
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) /
node_memory_MemTotal_bytes * 100

# Disk usage (%)
(node_filesystem_size_bytes - node_filesystem_free_bytes) /
node_filesystem_size_bytes * 100

# Latency p99 (dari histogram)
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# Aggregasi — total request semua endpoint
sum(rate(http_requests_total[5m]))

# Group by label
sum by (status)(rate(http_requests_total[5m]))

# Top 5 endpoint dengan request terbanyak
topk(5, sum by (endpoint)(rate(http_requests_total[5m])))
```

---

## 8.4 Alerting Rules

```yaml
# rules/alerts.yml
groups:
  - name: flask-app-alerts
    interval: 30s
    rules:
      # Alert jika aplikasi down
      - alert: AppDown
        expr: up{job="flask-app"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Aplikasi {{ $labels.instance }} down"
          description: "Tidak ada metrics dari {{ $labels.instance }} selama 1 menit."

      # Alert jika error rate tinggi
      - alert: HighErrorRate
        expr: |
          rate(http_requests_total{status=~"5.."}[5m]) /
          rate(http_requests_total[5m]) > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Error rate tinggi di {{ $labels.instance }}"
          description: "Error rate {{ $value | humanizePercentage }} selama 5 menit terakhir."

      # Alert jika latency tinggi (p99 > 1 detik)
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99,
            rate(http_request_duration_seconds_bucket[5m])
          ) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Latency p99 tinggi"
          description: "p99 latency: {{ $value }}s"

  - name: infrastructure-alerts
    rules:
      # CPU tinggi
      - alert: HighCPU
        expr: |
          100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "CPU tinggi di {{ $labels.instance }}"
          description: "CPU usage: {{ $value | humanize }}%"

      # Memory hampir penuh
      - alert: HighMemory
        expr: |
          (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) /
          node_memory_MemTotal_bytes * 100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Memory hampir penuh di {{ $labels.instance }}"

      # Disk hampir penuh
      - alert: DiskAlmostFull
        expr: |
          (node_filesystem_size_bytes - node_filesystem_free_bytes) /
          node_filesystem_size_bytes * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk hampir penuh di {{ $labels.instance }}: {{ $value | humanize }}%"
```

---

## 8.5 Alertmanager — Kirim Notifikasi

```yaml
# alertmanager.yml
global:
  smtp_smarthost: "smtp.gmail.com:587"
  smtp_from: "alerts@example.com"
  smtp_auth_username: "alerts@example.com"
  smtp_auth_password: "app-password"

# Template notifikasi
templates:
  - "/etc/alertmanager/templates/*.tmpl"

# Routing — siapa menerima alert apa
route:
  group_by: [alertname, severity]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: default-receiver

  routes:
    # Critical alerts → PagerDuty
    - match:
        severity: critical
      receiver: pagerduty
      continue: true

    # Infrastructure alerts → Slack #ops
    - match_re:
        alertname: "(HighCPU|HighMemory|DiskAlmostFull)"
      receiver: slack-infra

# Receivers
receivers:
  - name: default-receiver
    email_configs:
      - to: "team@example.com"
        require_tls: true
        send_resolved: true

  - name: slack-infra
    slack_configs:
      - api_url: "https://hooks.slack.com/services/xxx/yyy/zzz"
        channel: "#ops-alerts"
        title: "[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}"
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Severity:* {{ .Labels.severity }}
          *Details:* {{ .Annotations.description }}
          {{ end }}
        send_resolved: true

  - name: pagerduty
    pagerduty_configs:
      - routing_key: "your-pagerduty-integration-key"
        description: "{{ .CommonAnnotations.summary }}"

# Inhibit rules — jangan kirim alert minor jika major sudah aktif
inhibit_rules:
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: [alertname, instance]
```

---

## 8.6 Grafana Dashboard

### Konsep Dashboard Grafana

```
Dashboard
└── Row (pengelompokan panel)
    ├── Panel (visualisasi tunggal)
    │   ├── Stat (angka tunggal, contoh: uptime)
    │   ├── Time Series (grafik garis waktu)
    │   ├── Gauge (speedometer)
    │   ├── Table (tabel data)
    │   ├── Heatmap (pola waktu-ke-waktu)
    │   ├── Bar Chart
    │   └── Alert List
    └── Panel ...
```

### Contoh Query Dashboard

```
# Panel: Request Rate (req/s)
sum(rate(http_requests_total{job="flask-app"}[1m]))

# Panel: Error Rate (%)
sum(rate(http_requests_total{status=~"5..", job="flask-app"}[1m])) /
sum(rate(http_requests_total{job="flask-app"}[1m])) * 100

# Panel: Latency p50/p90/p99
histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[1m]))
histogram_quantile(0.90, rate(http_request_duration_seconds_bucket[1m]))
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[1m]))

# Panel: CPU Usage per Node
100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)

# Panel: Memory Usage
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) /
node_memory_MemTotal_bytes * 100

# Panel: Active Kubernetes Pods
kube_deployment_status_replicas_available{namespace="devops-lab"}
```

---

## 8.7 Logging Terpusat

### Stack Logging

```
Application Logs
      ↓
  Promtail/Fluentd/Filebeat    (log shipper)
      ↓
  Loki / Elasticsearch         (log storage)
      ↓
  Grafana / Kibana             (visualisasi & search)
```

### Loki + Promtail (Ringan, cocok dengan Grafana)

```yaml
# docker-compose-loki.yml
services:
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    volumes:
      - ./loki-config.yml:/etc/loki/local-config.yaml
      - loki_data:/loki
    command: -config.file=/etc/loki/local-config.yaml

  promtail:
    image: grafana/promtail:latest
    volumes:
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./promtail-config.yml:/etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml
```

### Query LogQL (Loki)

```
# Tampilkan semua log dari app flask-app
{job="flask-app"}

# Filter log error
{job="flask-app"} |= "ERROR"

# Regex filter
{job="flask-app"} |~ "5[0-9][0-9]"

# Parse log JSON
{job="flask-app"} | json | level="error"

# Rate log error per menit
rate({job="flask-app"} |= "ERROR" [1m])

# Count log per level
sum by (level) (count_over_time({job="flask-app"} | json | __error__="" [5m]))
```

---

## 8.8 USE dan RED Method

### USE Method (Infrastructure)

| Metrik | Penjelasan |
|---|---|
| **U**tilization | Seberapa sibuk resource? (CPU %, Memory %) |
| **S**aturation | Apakah ada antrian/backlog? |
| **E**rrors | Berapa banyak error? |

### RED Method (Services/API)

| Metrik | Penjelasan |
|---|---|
| **R**ate | Berapa request per detik? |
| **E**rrors | Berapa persen request yang error? |
| **D**uration | Berapa lama request diproses? |

---

## Ringkasan

| Komponen | Tools | Fungsi |
|---|---|---|
| Metrics | Prometheus + Node Exporter | Kumpul & simpan time-series data |
| Visualization | Grafana | Dashboard & alerting UI |
| Alerting | Alertmanager | Routing & notifikasi alert |
| Logging | Loki + Promtail | Log aggregation ringan |
| Tracing | Jaeger / Tempo | Distributed tracing |
| All-in-one K8s | kube-prometheus-stack | Prometheus + Grafana + rules siap pakai |
