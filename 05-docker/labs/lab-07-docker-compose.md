# LAB 07 — Docker Compose: Multi-Container Application

## Tujuan
Menjalankan aplikasi multi-container (Flask + PostgreSQL + Redis + NGINX) menggunakan Docker Compose dengan konfigurasi production-ready.

## Prasyarat
- Lab 06 selesai
- Docker Compose tersedia (`docker compose version`)

## Estimasi Waktu
60 menit

---

## Skenario

Anda akan membangun stack aplikasi lengkap:

```
Internet
    │
    ▼
nginx:80  (Reverse Proxy + Static Files)
    │
    ├── /api/* → flask-app:8080 (REST API)
    │               │
    │               ├── PostgreSQL:5432 (Database)
    │               └── Redis:6379 (Cache)
    │
    └── /* → Static HTML (via NGINX)
```

---

## Langkah 1: Siapkan Struktur Proyek

```bash
mkdir -p ~/devops-labs/docker/compose-stack
cd ~/devops-labs/docker/compose-stack

# Struktur direktori
mkdir -p app nginx/conf.d static db
```

---

## Langkah 2: Buat Aplikasi Flask dengan Database

```bash
cat > app/app.py << 'EOF'
from flask import Flask, jsonify, request
import os
import redis
import psycopg2
import datetime
import json

app = Flask(__name__)

# Konfigurasi dari environment
REDIS_URL = os.getenv("REDIS_URL", "redis://localhost:6379")
DATABASE_URL = os.getenv("DATABASE_URL", "")
APP_VERSION = os.getenv("APP_VERSION", "1.0.0")

def get_redis():
    try:
        r = redis.from_url(REDIS_URL, decode_responses=True)
        r.ping()
        return r
    except Exception as e:
        return None

def get_db():
    try:
        conn = psycopg2.connect(DATABASE_URL)
        return conn
    except Exception as e:
        return None

@app.route("/api/")
def index():
    return jsonify({
        "service": "DevOps Training API",
        "version": APP_VERSION,
        "status": "running",
        "timestamp": datetime.datetime.utcnow().isoformat()
    })

@app.route("/api/health")
def health():
    redis_ok = get_redis() is not None
    db_ok = get_db() is not None
    
    status = "healthy" if redis_ok and db_ok else "degraded"
    code = 200 if status == "healthy" else 503
    
    return jsonify({
        "status": status,
        "services": {
            "redis": "ok" if redis_ok else "error",
            "database": "ok" if db_ok else "error"
        }
    }), code

@app.route("/api/cache/set", methods=["POST"])
def cache_set():
    data = request.get_json()
    r = get_redis()
    if not r:
        return jsonify({"error": "Redis tidak tersedia"}), 503
    
    key = data.get("key", "test")
    value = data.get("value", "hello")
    r.setex(key, 300, json.dumps(value))
    return jsonify({"message": f"Key '{key}' tersimpan (TTL: 300s)"})

@app.route("/api/cache/get/<key>")
def cache_get(key):
    r = get_redis()
    if not r:
        return jsonify({"error": "Redis tidak tersedia"}), 503
    
    value = r.get(key)
    if value:
        return jsonify({"key": key, "value": json.loads(value), "source": "cache"})
    return jsonify({"key": key, "value": None, "source": "cache"}), 404

@app.route("/api/db/test")
def db_test():
    conn = get_db()
    if not conn:
        return jsonify({"error": "Database tidak tersedia"}), 503
    
    cur = conn.cursor()
    cur.execute("SELECT version(), NOW()")
    version, now = cur.fetchone()
    cur.close()
    conn.close()
    
    return jsonify({
        "database": "PostgreSQL",
        "version": version.split(",")[0],
        "server_time": str(now)
    })

if __name__ == "__main__":
    port = int(os.getenv("PORT", 8080))
    app.run(host="0.0.0.0", port=port, debug=False)
EOF

cat > app/requirements.txt << 'EOF'
flask==3.0.3
gunicorn==22.0.0
redis==5.0.4
psycopg2-binary==2.9.9
EOF

cat > app/Dockerfile << 'EOF'
FROM python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
    CMD curl -f http://localhost:8080/api/health || exit 1

CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "2", "app:app"]
EOF
```

---

## Langkah 3: Konfigurasi NGINX

```bash
cat > nginx/conf.d/default.conf << 'EOF'
upstream flask_app {
    server app:8080;
}

server {
    listen 80;
    server_name localhost;

    access_log /var/log/nginx/access.log combined;
    error_log  /var/log/nginx/error.log warn;

    # Proxy ke Flask API
    location /api/ {
        proxy_pass http://flask_app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 10s;
        proxy_read_timeout 30s;
        proxy_buffering off;
    }

    # Static files
    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri $uri/ /index.html;
    }

    # Health check NGINX sendiri
    location /nginx-health {
        access_log off;
        return 200 "ok\n";
        add_header Content-Type text/plain;
    }
}
EOF

# Buat halaman statis
cat > static/index.html << 'EOF'
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>DevOps Training Stack</title>
    <style>
        body { font-family: -apple-system, sans-serif; max-width: 800px; margin: 50px auto; padding: 20px; }
        .card { background: #f5f5f5; border-radius: 8px; padding: 20px; margin: 20px 0; }
        .badge { background: #4CAF50; color: white; padding: 2px 8px; border-radius: 4px; font-size: 12px; }
        pre { background: #272822; color: #f8f8f2; padding: 15px; border-radius: 4px; overflow-x: auto; }
        button { background: #2196F3; color: white; border: none; padding: 10px 20px; border-radius: 4px; cursor: pointer; }
        button:hover { background: #1976D2; }
        #result { margin-top: 10px; }
    </style>
</head>
<body>
    <h1>DevOps Training Stack <span class="badge">v1.0.0</span></h1>
    <p>Stack multi-container dengan Docker Compose</p>

    <div class="card">
        <h2>Arsitektur</h2>
        <pre>
Internet → NGINX:80
               ├── /api/* → Flask:8080
               │               ├── PostgreSQL:5432
               │               └── Redis:6379
               └── /*     → Static HTML
        </pre>
    </div>

    <div class="card">
        <h2>Test API Endpoints</h2>
        <button onclick="testApi('/api/')">GET /api/</button>
        <button onclick="testApi('/api/health')">Health Check</button>
        <button onclick="testApi('/api/db/test')">Test Database</button>
        <div id="result"></div>
    </div>

    <script>
        async function testApi(url) {
            const res = document.getElementById('result');
            res.innerHTML = 'Loading...';
            try {
                const response = await fetch(url);
                const data = await response.json();
                res.innerHTML = '<pre>' + JSON.stringify(data, null, 2) + '</pre>';
            } catch(e) {
                res.innerHTML = '<p style="color:red">Error: ' + e.message + '</p>';
            }
        }
    </script>
</body>
</html>
EOF
```

---

## Langkah 4: Buat docker-compose.yml

```bash
cat > docker-compose.yml << 'EOF'
services:

  # ─── NGINX Reverse Proxy ──────────────────────────────
  nginx:
    image: nginx:1.25-alpine
    container_name: compose-nginx
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./static:/usr/share/nginx/html:ro
    depends_on:
      app:
        condition: service_healthy
    networks:
      - frontend
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost/nginx-health"]
      interval: 30s
      timeout: 5s
      retries: 3

  # ─── Flask Application ────────────────────────────────
  app:
    build:
      context: ./app
      dockerfile: Dockerfile
    container_name: compose-app
    restart: unless-stopped
    expose:
      - "8080"
    environment:
      - APP_VERSION=1.0.0
      - APP_ENV=production
      - PORT=8080
      - DATABASE_URL=postgresql://devops:Training123@db:5432/devopsdb
      - REDIS_URL=redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_healthy
    networks:
      - frontend
      - backend
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/api/health"]
      interval: 30s
      timeout: 10s
      start_period: 15s
      retries: 3

  # ─── PostgreSQL Database ──────────────────────────────
  db:
    image: postgres:16-alpine
    container_name: compose-db
    restart: unless-stopped
    environment:
      POSTGRES_DB: devopsdb
      POSTGRES_USER: devops
      POSTGRES_PASSWORD: Training123
    volumes:
      - pg_data:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/01-init.sql:ro
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U devops -d devopsdb"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

  # ─── Redis Cache ──────────────────────────────────────
  cache:
    image: redis:7-alpine
    container_name: compose-cache
    restart: unless-stopped
    command: >
      redis-server
      --maxmemory 128mb
      --maxmemory-policy allkeys-lru
      --save 60 1
    volumes:
      - redis_data:/data
    networks:
      - backend
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  pg_data:
    driver: local
  redis_data:
    driver: local

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true    # Backend network tidak terekspos ke host
EOF

# Buat SQL inisialisasi database
cat > db/init.sql << 'EOF'
-- Inisialisasi database DevOps Training
CREATE TABLE IF NOT EXISTS events (
    id SERIAL PRIMARY KEY,
    event_type VARCHAR(50) NOT NULL,
    message TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

INSERT INTO events (event_type, message) VALUES
    ('startup', 'Database berhasil diinisialisasi'),
    ('info', 'DevOps Training Stack v1.0.0');

GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO devops;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO devops;
EOF
```

---

## Langkah 5: Jalankan Stack

```bash
# Build dan start semua service
docker compose up -d --build

# Pantau progress startup
docker compose ps

# Tunggu hingga semua healthy
echo "Menunggu semua service healthy..."
sleep 15
docker compose ps
```

Output yang diharapkan:
```
NAME             IMAGE                 STATUS
compose-app      compose-stack-app     Up (healthy)
compose-cache    redis:7-alpine        Up (healthy)
compose-db       postgres:16-alpine    Up (healthy)
compose-nginx    nginx:1.25-alpine     Up (healthy)
```

---

## Langkah 6: Test Stack Lengkap

```bash
echo "=== Test Halaman Statis ==="
curl -s http://localhost/ | grep -o '<title>.*</title>'

echo ""
echo "=== Test API Root ==="
curl -s http://localhost/api/ | python3 -m json.tool

echo ""
echo "=== Test Health Check ==="
curl -s http://localhost/api/health | python3 -m json.tool

echo ""
echo "=== Test Database ==="
curl -s http://localhost/api/db/test | python3 -m json.tool

echo ""
echo "=== Test Redis Cache ==="
# Set cache
curl -s -X POST http://localhost/api/cache/set \
  -H "Content-Type: application/json" \
  -d '{"key": "greeting", "value": "Hello DevOps!"}' | python3 -m json.tool

# Get cache
curl -s http://localhost/api/cache/get/greeting | python3 -m json.tool
```

---

## Langkah 7: Lihat Log Semua Service

```bash
# Log semua service
docker compose logs

# Log service tertentu
docker compose logs app
docker compose logs db
docker compose logs nginx

# Follow log real-time (semua service)
docker compose logs -f

# Follow log service tertentu
docker compose logs -f app &
LOG_PID=$!

# Kirim request untuk melihat log
curl -s http://localhost/api/ > /dev/null
curl -s http://localhost/api/health > /dev/null

sleep 2
kill $LOG_PID 2>/dev/null
```

---

## Langkah 8: Simulasi Kegagalan Service

```bash
# Matikan service cache
docker compose stop cache

# Test health — harus degraded
curl -s http://localhost/api/health | python3 -m json.tool

# Hidupkan kembali
docker compose start cache

# Tunggu hingga healthy
sleep 10
curl -s http://localhost/api/health | python3 -m json.tool
```

---

## Langkah 9: Scale Application

```bash
# Scale app ke 3 instance
docker compose up -d --scale app=3

# Cek running instances
docker compose ps

# Perhatikan: NGINX otomatis load balance ke semua instance
# (karena menggunakan nama service "app" di upstream)
for i in {1..6}; do
    curl -s http://localhost/api/ | python3 -c "
import sys, json
d = json.load(sys.stdin)
print(f'Response dari: {d}')
"
done
```

> Catatan: Untuk scale bekerja optimal dengan NGINX, gunakan `expose` bukan `ports` di service `app`.

---

## Langkah 10: Override dengan docker-compose.override.yml

```bash
# File override untuk development
cat > docker-compose.override.yml << 'EOF'
services:
  app:
    environment:
      - APP_ENV=development
    volumes:
      - ./app:/app:delegated    # Hot reload kode
    command: ["python3", "-m", "flask", "run", "--host=0.0.0.0", "--port=8080", "--debug"]

  db:
    ports:
      - "5432:5432"             # Expose DB untuk akses langsung

  cache:
    ports:
      - "6379:6379"             # Expose Redis untuk akses langsung
EOF

# docker compose otomatis baca override file
docker compose up -d
docker compose ps
```

---

## Cleanup

```bash
# Stop dan hapus semua container
docker compose down

# Hapus juga volume (data akan hilang!)
docker compose down -v

# Hapus juga image yang dibangun
docker compose down -v --rmi local

echo "Stack berhasil dihapus"
```

---

## Verifikasi Akhir

```bash
cd ~/devops-labs/docker/compose-stack

docker compose up -d --build
sleep 20

echo "=== Verifikasi Lab 07 ==="
echo -n "NGINX berjalan: "
  docker compose ps nginx | grep -q "healthy" && echo "✅" || echo "⏳ (menunggu)"
echo -n "App berjalan: "
  docker compose ps app | grep -q "healthy" && echo "✅" || echo "⏳ (menunggu)"
echo -n "Database berjalan: "
  docker compose ps db | grep -q "healthy" && echo "✅" || echo "⏳ (menunggu)"
echo -n "Redis berjalan: "
  docker compose ps cache | grep -q "healthy" && echo "✅" || echo "⏳ (menunggu)"
echo -n "API response OK: "
  curl -sf http://localhost/api/ | grep -q "running" && echo "✅" || echo "❌"
echo -n "Health check OK: "
  curl -sf http://localhost/api/health | grep -q "healthy" && echo "✅" || echo "❌"
echo -n "Database query OK: "
  curl -sf http://localhost/api/db/test | grep -q "PostgreSQL" && echo "✅" || echo "❌"

docker compose down -v
```

---

## Checkpoint ✅

- [ ] Stack 4 service berjalan (NGINX, Flask, PostgreSQL, Redis)
- [ ] Semua service memiliki healthcheck dan status `healthy`
- [ ] Request melalui NGINX berhasil diteruskan ke Flask
- [ ] Flask berhasil terhubung ke PostgreSQL dan Redis
- [ ] Memahami `depends_on` dengan `condition: service_healthy`
- [ ] Berhasil mensimulasikan kegagalan dan recovery service
- [ ] Memahami internal network (backend tidak terekspos)
