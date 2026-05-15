# LAB 06 — Menulis Dockerfile

## Tujuan
Membuat Dockerfile untuk aplikasi Python Flask, menerapkan multi-stage build, dan memverifikasi image berjalan dengan benar.

## Prasyarat
- Docker terinstall dan berjalan (Lab 01)
- Dasar Python (tidak perlu ahli)

## Estimasi Waktu
45 menit

---

## Skenario

Anda diminta untuk mengkontainerisasi aplikasi REST API Python Flask yang sebelumnya hanya bisa dijalankan di laptop developer. Target akhir: satu perintah `docker run` dan aplikasi langsung siap pakai.

---

## Langkah 1: Buat Aplikasi Flask

```bash
mkdir -p ~/devops-labs/docker/flask-app
cd ~/devops-labs/docker/flask-app
```

Buat file aplikasi:

```bash
cat > app.py << 'EOF'
from flask import Flask, jsonify
import os
import datetime

app = Flask(__name__)

APP_VERSION = os.getenv("APP_VERSION", "1.0.0")
APP_ENV = os.getenv("APP_ENV", "development")

@app.route("/")
def index():
    return jsonify({
        "app": "DevOps Training API",
        "version": APP_VERSION,
        "environment": APP_ENV,
        "status": "running"
    })

@app.route("/health")
def health():
    return jsonify({
        "status": "healthy",
        "timestamp": datetime.datetime.utcnow().isoformat()
    }), 200

@app.route("/info")
def info():
    return jsonify({
        "hostname": os.uname().nodename,
        "python_version": os.popen("python3 --version").read().strip(),
        "uptime": "N/A"
    })

if __name__ == "__main__":
    port = int(os.getenv("PORT", 8080))
    debug = APP_ENV == "development"
    app.run(host="0.0.0.0", port=port, debug=debug)
EOF

cat > requirements.txt << 'EOF'
flask==3.0.3
gunicorn==22.0.0
EOF

cat > gunicorn.conf.py << 'EOF'
# Gunicorn configuration
bind = "0.0.0.0:8080"
workers = 2
worker_class = "sync"
worker_connections = 1000
timeout = 30
keepalive = 2
accesslog = "-"
errorlog = "-"
loglevel = "info"
EOF

# Buat .dockerignore
cat > .dockerignore << 'EOF'
__pycache__
*.pyc
*.pyo
.env
.git
.gitignore
*.log
venv/
.venv/
tests/
*.md
EOF
```

---

## Langkah 2: Dockerfile Sederhana (Versi 1)

```bash
cat > Dockerfile.v1 << 'EOF'
FROM python:3.12

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

EXPOSE 8080

CMD ["python3", "app.py"]
EOF
```

Build dan cek ukurannya:

```bash
docker build -t flask-app:v1 -f Dockerfile.v1 .
docker images flask-app:v1
```

Catat ukuran image (biasanya >1 GB untuk `python:3.12`).

---

## Langkah 3: Dockerfile Optimized (Versi 2 — Slim)

```bash
cat > Dockerfile.v2 << 'EOF'
FROM python:3.12-slim

# Install dependency sistem minimal
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Salin requirements dulu (manfaatkan layer cache)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Salin kode aplikasi
COPY app.py .
COPY gunicorn.conf.py .

# Buat user non-root (keamanan)
RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser

EXPOSE 8080

# Healthcheck
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# Gunakan Gunicorn untuk production
CMD ["gunicorn", "--config", "gunicorn.conf.py", "app:app"]
EOF
```

Build dan bandingkan:

```bash
docker build -t flask-app:v2 -f Dockerfile.v2 .
docker images flask-app
```

---

## Langkah 4: Multi-Stage Build (Versi 3 — Production Ready)

```bash
cat > Dockerfile << 'EOF'
# ─── Stage 1: Builder ────────────────────────────────────────
FROM python:3.12-slim AS builder

WORKDIR /build

RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .

# Install ke direktori lokal (bukan sistem)
RUN pip install --user --no-cache-dir -r requirements.txt

# ─── Stage 2: Production ─────────────────────────────────────
FROM python:3.12-slim AS production

ARG APP_VERSION=1.0.0
ARG BUILD_DATE

# Metadata image
LABEL org.opencontainers.image.title="DevOps Training Flask API" \
      org.opencontainers.image.version="${APP_VERSION}" \
      org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.authors="devops-team@example.com"

# Install curl untuk healthcheck saja
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Salin Python packages dari builder
COPY --from=builder /root/.local /home/appuser/.local

# Salin kode aplikasi
COPY app.py gunicorn.conf.py ./

# Variabel environment
ENV APP_VERSION=${APP_VERSION} \
    APP_ENV=production \
    PORT=8080 \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PATH=/home/appuser/.local/bin:$PATH

# User non-root
RUN groupadd -r appuser && useradd -r -g appuser appuser \
    && chown -R appuser:appuser /app
USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:${PORT}/health || exit 1

ENTRYPOINT ["gunicorn"]
CMD ["--config", "gunicorn.conf.py", "app:app"]
EOF
```

Build dengan argumen:

```bash
docker build \
  --build-arg APP_VERSION=1.0.0 \
  --build-arg BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
  -t flask-app:latest \
  -t flask-app:1.0.0 \
  .

# Lihat semua versi
docker images flask-app
```

---

## Langkah 5: Jalankan dan Test

```bash
# Jalankan container
docker run -d \
  --name flask-api \
  -p 8080:8080 \
  -e APP_ENV=production \
  -e APP_VERSION=1.0.0 \
  flask-app:latest

# Cek berjalan
docker ps | grep flask-api

# Test endpoints
echo "=== GET / ==="
curl -s http://localhost:8080/ | python3 -m json.tool

echo ""
echo "=== GET /health ==="
curl -s http://localhost:8080/health | python3 -m json.tool

echo ""
echo "=== GET /info ==="
curl -s http://localhost:8080/info | python3 -m json.tool

# Cek healthcheck status
docker inspect flask-api --format '{{.State.Health.Status}}'
```

---

## Langkah 6: Analisis Image

```bash
# Lihat history layer image
docker history flask-app:latest
docker history --no-trunc flask-app:latest

# Inspeksi metadata
docker inspect flask-app:latest | python3 -m json.tool | head -80

# Perbandingan ukuran semua versi
echo "=== Perbandingan Ukuran Image ==="
docker images flask-app --format "table {{.Tag}}\t{{.Size}}"

# Dive tool untuk analisis layer (opsional, install terpisah)
# docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock \
#   wagoodman/dive flask-app:latest
```

---

## Langkah 7: Log dan Monitoring Container

```bash
# Lihat log container
docker logs flask-api
docker logs -f flask-api      # Follow

# Statistik resource
docker stats flask-api --no-stream

# Masuk ke container (debugging)
docker exec -it flask-api sh

# Di dalam container:
# ps aux                # lihat proses
# env                   # lihat environment variable
# curl http://localhost:8080/health  # test dari dalam
# exit
```

---

## Langkah 8: Test Layer Cache

Uji manfaat layer cache saat kode berubah:

```bash
# Ubah hanya kode app (bukan requirements)
echo "" >> app.py
echo "# Komentar tambahan" >> app.py

# Build ulang — hanya layer COPY app.py yang re-build
time docker build -t flask-app:v1.0.1 .

# Ubah requirements (simulasi tambah dependency)
echo "requests==2.31.0" >> requirements.txt

# Build ulang — layer pip install + semua berikutnya harus rebuild
time docker build -t flask-app:v1.0.2 .
```

Bandingkan waktu build — yang pertama lebih cepat karena cache `pip install`.

---

## Langkah 9: Push ke Registry (Opsional)

```bash
# Login ke Docker Hub
docker login

# Tag untuk Docker Hub
docker tag flask-app:latest YOUR_USERNAME/flask-app:latest
docker tag flask-app:1.0.0 YOUR_USERNAME/flask-app:1.0.0

# Push
docker push YOUR_USERNAME/flask-app:latest
docker push YOUR_USERNAME/flask-app:1.0.0

# Verifikasi: coba pull dari image yang sudah push
docker rmi YOUR_USERNAME/flask-app:latest
docker pull YOUR_USERNAME/flask-app:latest
docker run --rm -p 8081:8080 YOUR_USERNAME/flask-app:latest
```

---

## Cleanup

```bash
docker stop flask-api
docker rm flask-api
docker rmi flask-app:v1 flask-app:v2 flask-app:latest flask-app:1.0.0 \
           flask-app:v1.0.1 flask-app:v1.0.2 2>/dev/null
echo "Cleanup selesai"
```

---

## Verifikasi Akhir

```bash
cd ~/devops-labs/docker/flask-app

# Build fresh untuk verifikasi
docker build -t flask-app:verify .

# Run + test
CID=$(docker run -d -p 8080:8080 flask-app:verify)
sleep 3

echo "=== Verifikasi Lab 06 ==="
echo -n "Container berjalan: "
  docker ps | grep -q $CID && echo "✅" || echo "❌"
echo -n "Endpoint / response OK: "
  curl -sf http://localhost:8080/ | grep -q "running" && echo "✅" || echo "❌"
echo -n "Endpoint /health OK: "
  curl -sf http://localhost:8080/health | grep -q "healthy" && echo "✅" || echo "❌"
echo -n "Berjalan sebagai non-root: "
  docker exec $CID whoami | grep -q "appuser" && echo "✅" || echo "❌"

docker stop $CID && docker rm $CID
docker rmi flask-app:verify
```

---

## Checkpoint ✅

- [ ] Aplikasi Flask berhasil berjalan di container
- [ ] Memahami perbedaan image `python:3.12` vs `python:3.12-slim`
- [ ] Multi-stage build berhasil menghasilkan image yang lebih kecil
- [ ] Healthcheck aktif dan menampilkan status `healthy`
- [ ] Container berjalan sebagai user non-root
- [ ] Memahami manfaat layer cache saat rebuild
- [ ] Memahami arti setiap instruksi Dockerfile (FROM, RUN, COPY, ENV, dll)
