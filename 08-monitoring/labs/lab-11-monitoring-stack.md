# LAB 11 — Monitoring Stack: Prometheus + Grafana

## Tujuan
Membangun monitoring stack lengkap menggunakan Docker Compose: Prometheus scraping metrics aplikasi Flask, Grafana untuk visualisasi, dan Alertmanager untuk notifikasi.

## Prasyarat
- Docker dan Docker Compose terinstall (Lab 01)
- Lab 07 (Docker Compose) selesai

## Estimasi Waktu
60 menit

---

## Skenario

Anda akan membangun monitoring stack untuk memantau aplikasi Flask API beserta infrastruktur host. Dashboard Grafana akan menampilkan request rate, error rate, dan latency secara real-time.

---

## Langkah 1: Siapkan Struktur Proyek

```bash
mkdir -p ~/devops-labs/monitoring
cd ~/devops-labs/monitoring

# Struktur direktori
mkdir -p {prometheus,grafana/{provisioning/{datasources,dashboards},dashboards},alertmanager,app}
tree .
```

---

## Langkah 2: Buat Aplikasi Flask dengan Prometheus Metrics

```bash
cat > app/app.py << 'EOF'
from flask import Flask, jsonify, request
from prometheus_client import Counter, Histogram, Gauge, generate_latest, CONTENT_TYPE_LATEST
import time
import os
import random

app = Flask(__name__)

# ─── Metrics Definitions ────────────────────────
REQUEST_COUNT = Counter(
    "http_requests_total",
    "Total HTTP requests",
    ["method", "endpoint", "status"]
)

REQUEST_LATENCY = Histogram(
    "http_request_duration_seconds",
    "HTTP request duration in seconds",
    ["method", "endpoint"],
    buckets=[0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
)

ACTIVE_REQUESTS = Gauge(
    "http_active_requests",
    "Number of active HTTP requests"
)

APP_INFO = Gauge(
    "app_info",
    "Application info",
    ["version", "environment"]
)

# Set app info metric
APP_INFO.labels(
    version=os.getenv("APP_VERSION", "1.0.0"),
    environment=os.getenv("APP_ENV", "development")
).set(1)


# ─── Middleware: Catat metrics setiap request ────
@app.before_request
def before_request():
    request.start_time = time.time()
    ACTIVE_REQUESTS.inc()


@app.after_request
def after_request(response):
    ACTIVE_REQUESTS.dec()
    duration = time.time() - request.start_time

    # Jangan catat endpoint /metrics itu sendiri
    if request.path != "/metrics":
        REQUEST_COUNT.labels(
            method=request.method,
            endpoint=request.path,
            status=response.status_code
        ).inc()

        REQUEST_LATENCY.labels(
            method=request.method,
            endpoint=request.path
        ).observe(duration)

    return response


# ─── Endpoints ──────────────────────────────────
@app.route("/")
def index():
    return jsonify({
        "service": "DevOps Monitoring Demo",
        "version": os.getenv("APP_VERSION", "1.0.0"),
        "status": "running"
    })


@app.route("/health")
def health():
    return jsonify({"status": "healthy"}), 200


@app.route("/api/users")
def get_users():
    # Simulasi latency variabel
    time.sleep(random.uniform(0.01, 0.1))
    return jsonify({"users": ["alice", "bob", "carol"], "count": 3})


@app.route("/api/orders")
def get_orders():
    # Simulasi latency lebih tinggi (DB query lambat)
    time.sleep(random.uniform(0.05, 0.5))
    return jsonify({"orders": [{"id": 1, "item": "Laptop"}, {"id": 2, "item": "Mouse"}]})


@app.route("/api/error")
def trigger_error():
    # Simulasi error 20% dari request
    if random.random() < 0.2:
        return jsonify({"error": "Internal Server Error"}), 500
    return jsonify({"message": "OK"})


@app.route("/api/slow")
def slow_endpoint():
    # Simulasi endpoint lambat
    delay = random.uniform(0.5, 3.0)
    time.sleep(delay)
    return jsonify({"message": f"Selesai setelah {delay:.2f}s"})


# ─── Prometheus metrics endpoint ─────────────────
@app.route("/metrics")
def metrics():
    return generate_latest(), 200, {"Content-Type": CONTENT_TYPE_LATEST}


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080, debug=False)
EOF

cat > app/requirements.txt << 'EOF'
flask==3.0.3
gunicorn==22.0.0
prometheus-client==0.20.0
EOF

cat > app/Dockerfile << 'EOF'
FROM python:3.12-slim
RUN apt-get update && apt-get install -y --no-install-recommends curl && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
RUN groupadd -r app && useradd -r -g app app
USER app
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s \
    CMD curl -f http://localhost:8080/health || exit 1
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "2", "app:app"]
EOF
```

---

## Langkah 3: Konfigurasi Prometheus

```bash
cat > prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  scrape_timeout: 10s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

rule_files:
  - "rules/*.yml"

scrape_configs:
  # Prometheus scrape dirinya sendiri
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]
    metrics_path: /metrics

  # Scrape Flask App
  - job_name: flask-app
    scrape_interval: 10s
    static_configs:
      - targets: ["app:8080"]
    metrics_path: /metrics
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        replacement: flask-app

  # Scrape Node Exporter
  - job_name: node
    scrape_interval: 15s
    static_configs:
      - targets: ["node-exporter:9100"]
EOF

# Buat alert rules
mkdir -p prometheus/rules

cat > prometheus/rules/app-alerts.yml << 'EOF'
groups:
  - name: flask-app
    rules:
      - alert: AppDown
        expr: up{job="flask-app"} == 0
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "Flask App down"
          description: "Flask App tidak merespons selama 30 detik."

      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status="500"}[5m])) /
          sum(rate(http_requests_total[5m])) > 0.1
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Error rate tinggi"
          description: "Error rate {{ $value | humanizePercentage }} dalam 5 menit terakhir."

      - alert: SlowResponse
        expr: |
          histogram_quantile(0.95,
            rate(http_request_duration_seconds_bucket[5m])
          ) > 1
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Response time lambat (p95 > 1s)"
          description: "p95 latency: {{ $value }}s"

  - name: infrastructure
    rules:
      - alert: HighCPU
        expr: |
          100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CPU usage tinggi: {{ $value | humanize }}%"

      - alert: HighMemory
        expr: |
          (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) /
          node_memory_MemTotal_bytes * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Memory usage tinggi: {{ $value | humanize }}%"
EOF
```

---

## Langkah 4: Konfigurasi Alertmanager

```bash
cat > alertmanager/alertmanager.yml << 'EOF'
global:
  resolve_timeout: 5m

route:
  group_by: [alertname, severity]
  group_wait: 10s
  group_interval: 5m
  repeat_interval: 1h
  receiver: default

receivers:
  - name: default
    webhook_configs:
      - url: "http://webhook-logger:5001/alerts"
        send_resolved: true

inhibit_rules:
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: [alertname]
EOF
```

---

## Langkah 5: Konfigurasi Grafana

```bash
# Datasource Prometheus untuk Grafana
cat > grafana/provisioning/datasources/prometheus.yml << 'EOF'
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
    jsonData:
      httpMethod: POST
      timeInterval: "15s"
EOF

# Auto-provisioning dashboard dari file
cat > grafana/provisioning/dashboards/default.yml << 'EOF'
apiVersion: 1

providers:
  - name: DevOps Training
    orgId: 1
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: true
EOF

# Dashboard JSON untuk Flask App
cat > grafana/dashboards/flask-app.json << 'EOF'
{
  "title": "Flask App Monitoring",
  "uid": "flask-app-lab",
  "tags": ["flask", "devops-training"],
  "time": {"from": "now-30m", "to": "now"},
  "refresh": "10s",
  "panels": [
    {
      "id": 1,
      "title": "Request Rate (req/s)",
      "type": "stat",
      "gridPos": {"h": 4, "w": 6, "x": 0, "y": 0},
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{job=\"flask-app\"}[1m]))",
          "legendFormat": "req/s"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "reqps",
          "color": {"mode": "thresholds"},
          "thresholds": {
            "steps": [
              {"value": 0, "color": "green"},
              {"value": 50, "color": "yellow"},
              {"value": 100, "color": "red"}
            ]
          }
        }
      }
    },
    {
      "id": 2,
      "title": "Error Rate (%)",
      "type": "stat",
      "gridPos": {"h": 4, "w": 6, "x": 6, "y": 0},
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{job=\"flask-app\", status=\"500\"}[1m])) / sum(rate(http_requests_total{job=\"flask-app\"}[1m])) * 100",
          "legendFormat": "error %"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "color": {"mode": "thresholds"},
          "thresholds": {
            "steps": [
              {"value": 0, "color": "green"},
              {"value": 5, "color": "yellow"},
              {"value": 10, "color": "red"}
            ]
          }
        }
      }
    },
    {
      "id": 3,
      "title": "Request Rate per Endpoint",
      "type": "timeseries",
      "gridPos": {"h": 8, "w": 12, "x": 0, "y": 4},
      "targets": [
        {
          "expr": "sum by(endpoint)(rate(http_requests_total{job=\"flask-app\"}[1m]))",
          "legendFormat": "{{endpoint}}"
        }
      ]
    },
    {
      "id": 4,
      "title": "Response Time (p50/p90/p99)",
      "type": "timeseries",
      "gridPos": {"h": 8, "w": 12, "x": 12, "y": 4},
      "targets": [
        {
          "expr": "histogram_quantile(0.50, rate(http_request_duration_seconds_bucket{job=\"flask-app\"}[1m]))",
          "legendFormat": "p50"
        },
        {
          "expr": "histogram_quantile(0.90, rate(http_request_duration_seconds_bucket{job=\"flask-app\"}[1m]))",
          "legendFormat": "p90"
        },
        {
          "expr": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{job=\"flask-app\"}[1m]))",
          "legendFormat": "p99"
        }
      ]
    }
  ]
}
EOF
```

---

## Langkah 6: Buat docker-compose.yml

```bash
cat > docker-compose.yml << 'EOF'
services:

  # ─── Flask Application ────────────────────────
  app:
    build: ./app
    container_name: mon-app
    restart: unless-stopped
    expose:
      - "8080"
    environment:
      - APP_VERSION=1.0.0
      - APP_ENV=production
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 5s
      retries: 3
    networks:
      - monitoring
      - app

  # ─── Prometheus ───────────────────────────────
  prometheus:
    image: prom/prometheus:latest
    container_name: mon-prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/rules:/etc/prometheus/rules:ro
      - prometheus_data:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --storage.tsdb.retention.time=15d
      - --web.enable-lifecycle
      - --web.enable-admin-api
    networks:
      - monitoring
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:9090/-/healthy"]
      interval: 30s
      timeout: 5s
      retries: 3

  # ─── Node Exporter ───────────────────────────
  node-exporter:
    image: prom/node-exporter:latest
    container_name: mon-node-exporter
    restart: unless-stopped
    pid: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - --path.procfs=/host/proc
      - --path.rootfs=/rootfs
      - --path.sysfs=/host/sys
      - --collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)
    networks:
      - monitoring

  # ─── Alertmanager ────────────────────────────
  alertmanager:
    image: prom/alertmanager:latest
    container_name: mon-alertmanager
    restart: unless-stopped
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
      - alertmanager_data:/alertmanager
    command:
      - --config.file=/etc/alertmanager/alertmanager.yml
      - --storage.path=/alertmanager
    networks:
      - monitoring

  # ─── Grafana ─────────────────────────────────
  grafana:
    image: grafana/grafana:latest
    container_name: mon-grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=Training2024
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_ANALYTICS_REPORTING_ENABLED=false
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - ./grafana/dashboards:/var/lib/grafana/dashboards:ro
    depends_on:
      prometheus:
        condition: service_healthy
    networks:
      - monitoring
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 5s
      retries: 3

volumes:
  prometheus_data:
  grafana_data:
  alertmanager_data:

networks:
  monitoring:
    driver: bridge
  app:
    driver: bridge
EOF
```

---

## Langkah 7: Jalankan Stack

```bash
docker compose up -d --build

# Pantau startup
echo "Menunggu semua service healthy..."
sleep 30
docker compose ps
```

Output yang diharapkan:
```
NAME                  STATUS
mon-alertmanager      Up (healthy)
mon-app               Up (healthy)
mon-grafana           Up (healthy)
mon-node-exporter     Up
mon-prometheus        Up (healthy)
```

---

## Langkah 8: Verifikasi Prometheus

```bash
# Cek targets yang di-scrape
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | grep -E '"health"|"job"'

# Test query PromQL
curl -s "http://localhost:9090/api/v1/query?query=up" | \
  python3 -c "import json,sys; d=json.load(sys.stdin); [print(r['metric']['job'], r['value'][1]) for r in d['data']['result']]"

# Buka Prometheus UI di browser:
echo "Prometheus UI: http://localhost:9090"
echo "Alertmanager: http://localhost:9093"
```

---

## Langkah 9: Generate Traffic untuk Mengisi Metrics

```bash
# Script untuk generate traffic simulasi
cat > generate-traffic.sh << 'SCRIPT'
#!/bin/bash
echo "Generating traffic ke Flask app..."

APP_URL="http://localhost:8080"

for i in $(seq 1 200); do
    # Request normal
    curl -sf "$APP_URL/api/users" > /dev/null
    curl -sf "$APP_URL/api/orders" > /dev/null
    curl -sf "$APP_URL/" > /dev/null
    
    # Request yang kadang error (20% chance)
    curl -sf "$APP_URL/api/error" > /dev/null
    
    # Request slow (setiap 10 iterasi)
    if [ $((i % 10)) -eq 0 ]; then
        curl -sf "$APP_URL/api/slow" > /dev/null
        echo "Iteration $i/200 selesai"
    fi
    
    sleep 0.1
done
echo "Traffic generation selesai!"
SCRIPT

chmod +x generate-traffic.sh
./generate-traffic.sh &
TRAFFIC_PID=$!
echo "Traffic generator berjalan (PID: $TRAFFIC_PID)"
```

---

## Langkah 10: Eksplorasi Grafana

```bash
echo "Buka Grafana di: http://localhost:3000"
echo "Username: admin"
echo "Password: Training2024"
```

**Panduan eksplorasi di Grafana:**

1. **Login** dengan admin/Training2024
2. **Lihat dashboard** yang sudah auto-provisioned:
   - Home → Dashboards → DevOps Training → Flask App Monitoring
3. **Explore metrics** (menu Explore):
   - Query: `sum(rate(http_requests_total[1m]))`
   - Query: `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[1m]))`
4. **Buat panel baru**:
   - Dashboard → Add panel → Add new visualization
   - Data source: Prometheus
   - Query: `up`
5. **Lihat Alerts** (Alerting → Alert rules):
   - Rules dari Prometheus akan muncul jika ada yang triggered

---

## Langkah 11: Query PromQL Manual

```bash
# Langsung query ke Prometheus API
BASE="http://localhost:9090/api/v1"

echo "=== Request Rate per Endpoint ==="
curl -sg "$BASE/query?query=sum+by(endpoint)(rate(http_requests_total[1m]))" | \
  python3 -c "
import json, sys
d = json.load(sys.stdin)
for r in d['data']['result']:
    print(f\"{r['metric'].get('endpoint', 'N/A')}: {float(r['value'][1]):.4f} req/s\")
"

echo ""
echo "=== Error Rate ==="
curl -sg "$BASE/query?query=sum(rate(http_requests_total{status='500'}[5m]))/sum(rate(http_requests_total[5m]))*100" | \
  python3 -c "
import json, sys
d = json.load(sys.stdin)
result = d['data']['result']
if result:
    print(f\"Error rate: {float(result[0]['value'][1]):.2f}%\")
else:
    print('Belum ada data')
"

echo ""
echo "=== Latency p95 ==="
curl -sg "$BASE/query?query=histogram_quantile(0.95,rate(http_request_duration_seconds_bucket[5m]))" | \
  python3 -c "
import json, sys
d = json.load(sys.stdin)
result = d['data']['result']
if result:
    print(f\"p95 latency: {float(result[0]['value'][1]):.3f}s\")
else:
    print('Belum ada data')
"
```

---

## Cleanup

```bash
kill $TRAFFIC_PID 2>/dev/null
docker compose down -v
echo "Monitoring stack dihentikan dan data dibersihkan"
```

---

## Verifikasi Akhir

```bash
docker compose up -d

sleep 30

echo "=== Verifikasi Lab 11 ==="
echo -n "Prometheus running: "
  curl -sf http://localhost:9090/-/healthy && echo "✅" || echo "❌"
echo -n "Grafana running: "
  curl -sf http://localhost:3000/api/health | grep -q "ok" && echo "✅" || echo "❌"
echo -n "Alertmanager running: "
  curl -sf http://localhost:9093/-/healthy && echo "✅" || echo "❌"
echo -n "Flask app metrics: "
  curl -sf http://localhost:8080/metrics | grep -q "http_requests_total" && echo "✅" || echo "❌"
echo -n "Prometheus scraping Flask: "
  curl -sf "http://localhost:9090/api/v1/targets" | grep -q '"flask-app"' && echo "✅" || echo "❌"

docker compose down -v
```

---

## Checkpoint ✅

- [ ] Prometheus berhasil men-scrape metrics dari Flask app
- [ ] Node Exporter memberikan metrics infrastruktur host
- [ ] Dashboard Grafana menampilkan request rate dan latency
- [ ] Alert rules terkonfigurasi di Prometheus
- [ ] Memahami PromQL untuk query metrics
- [ ] Bisa membuat panel baru di Grafana
- [ ] Memahami perbedaan Counter, Gauge, dan Histogram
