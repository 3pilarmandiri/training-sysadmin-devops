# LAB 04 — NGINX Reverse Proxy

## Tujuan
Mengkonfigurasi NGINX sebagai reverse proxy yang meneruskan request ke dua backend aplikasi berbeda berdasarkan path URL.

## Prasyarat
- Lab 01 selesai (lingkungan siap)
- Akses sudo

## Estimasi Waktu
45 menit

---

## Skenario

Anda diminta menyiapkan infrastruktur untuk aplikasi microservices:

```
Internet → nginx:80
              ├── /api/*   → backend-api:8001 (Node.js/Python app)
              ├── /app/*   → frontend:8002 (React app sim)
              └── /        → halaman status NGINX
```

---

## Langkah 1: Install NGINX

```bash
sudo apt update
sudo apt install -y nginx

# Start dan enable NGINX
sudo systemctl enable --now nginx
sudo systemctl status nginx

# Verifikasi berjalan
curl -s http://localhost | grep -i nginx
```

---

## Langkah 2: Simulasikan Backend Services

Gunakan Python HTTP server sederhana sebagai backend palsu:

```bash
# Buat direktori untuk backend
mkdir -p ~/lab-nginx/api
mkdir -p ~/lab-nginx/frontend

# Buat response backend API
cat > ~/lab-nginx/api/index.html << 'EOF'
{
  "service": "backend-api",
  "port": 8001,
  "status": "running",
  "version": "1.0.0",
  "message": "Hello from Backend API!"
}
EOF

# Buat response frontend
cat > ~/lab-nginx/frontend/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>DevOps App</title></head>
<body>
  <h1>Frontend Application</h1>
  <p>Running on port 8002</p>
  <p>Served via NGINX Reverse Proxy</p>
</body>
</html>
EOF
```

Jalankan kedua backend di terminal terpisah (gunakan `tmux` atau dua terminal):

```bash
# Terminal 1: Backend API di port 8001
cd ~/lab-nginx/api
python3 -m http.server 8001 &
echo "Backend API PID: $!"

# Terminal 2: Frontend di port 8002
cd ~/lab-nginx/frontend
python3 -m http.server 8002 &
echo "Frontend PID: $!"

# Verifikasi keduanya berjalan
curl http://localhost:8001
curl http://localhost:8002
```

---

## Langkah 3: Buat Konfigurasi NGINX

```bash
# Hapus konfigurasi default
sudo rm -f /etc/nginx/sites-enabled/default

# Buat konfigurasi baru
sudo tee /etc/nginx/sites-available/devops-lab.conf << 'EOF'
# Upstream definitions
upstream backend_api {
    server 127.0.0.1:8001;
    keepalive 16;
}

upstream frontend_app {
    server 127.0.0.1:8002;
    keepalive 16;
}

server {
    listen 80;
    server_name localhost;

    # Log format dengan timing
    access_log /var/log/nginx/devops-lab.access.log combined;
    error_log  /var/log/nginx/devops-lab.error.log warn;

    # Halaman status server
    location = / {
        default_type text/html;
        return 200 '
        <html>
        <head><title>DevOps Proxy</title></head>
        <body>
          <h1>NGINX Reverse Proxy</h1>
          <ul>
            <li><a href="/api/">/api/ → Backend API (port 8001)</a></li>
            <li><a href="/app/">/app/ → Frontend App (port 8002)</a></li>
          </ul>
        </body>
        </html>';
    }

    # Proxy ke Backend API
    location /api/ {
        proxy_pass http://backend_api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeout settings
        proxy_connect_timeout 10s;
        proxy_send_timeout    30s;
        proxy_read_timeout    30s;

        # Buffer settings
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;

        # Header tambahan untuk debug
        add_header X-Proxy-By "NGINX";
        add_header X-Upstream "backend-api";
    }

    # Proxy ke Frontend App
    location /app/ {
        proxy_pass http://frontend_app/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        add_header X-Proxy-By "NGINX";
        add_header X-Upstream "frontend-app";
    }

    # Health check endpoint
    location /health {
        access_log off;
        return 200 '{"status":"healthy","proxy":"nginx"}\n';
        add_header Content-Type application/json;
    }

    # NGINX status page (untuk monitoring)
    location /nginx-status {
        stub_status on;
        allow 127.0.0.1;
        deny all;
    }

    # Block akses ke file tersembunyi
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }
}
EOF
```

---

## Langkah 4: Aktifkan dan Test Konfigurasi

```bash
# Enable konfigurasi
sudo ln -sf /etc/nginx/sites-available/devops-lab.conf \
            /etc/nginx/sites-enabled/devops-lab.conf

# Test syntax konfigurasi
sudo nginx -t
```

Output yang diharapkan:
```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

```bash
# Reload NGINX (tanpa downtime)
sudo systemctl reload nginx

# Verifikasi NGINX berjalan
sudo systemctl status nginx
```

---

## Langkah 5: Test Reverse Proxy

```bash
echo "=== Test Halaman Utama ==="
curl -s http://localhost/ | grep -o '<h1>.*</h1>'

echo ""
echo "=== Test Proxy ke Backend API ==="
curl -s http://localhost/api/
curl -sv http://localhost/api/ 2>&1 | grep -E "< X-|< HTTP"

echo ""
echo "=== Test Proxy ke Frontend App ==="
curl -s http://localhost/app/ | grep -o '<h1>.*</h1>'
curl -sv http://localhost/app/ 2>&1 | grep -E "< X-|< HTTP"

echo ""
echo "=== Test Health Check ==="
curl -s http://localhost/health

echo ""
echo "=== Test NGINX Status (hanya dari localhost) ==="
curl -s http://localhost/nginx-status
```

Output yang diharapkan dari header:
```
< HTTP/1.1 200 OK
< X-Proxy-By: NGINX
< X-Upstream: backend-api
```

---

## Langkah 6: Analisis Log

```bash
# Lihat log akses
sudo tail -20 /var/log/nginx/devops-lab.access.log

# Lihat log error
sudo tail -20 /var/log/nginx/devops-lab.error.log

# Monitor real-time (buka terminal terpisah)
sudo tail -f /var/log/nginx/devops-lab.access.log &

# Kirim beberapa request untuk mengisi log
for i in {1..5}; do
    curl -s http://localhost/api/ > /dev/null
    curl -s http://localhost/app/ > /dev/null
    curl -s http://localhost/health > /dev/null
done

# Format log: IP - user [time] "request" status bytes "referer" "agent"
```

---

## Langkah 7: Tambah Rate Limiting

```bash
# Edit konfigurasi untuk tambah rate limiting
sudo tee /etc/nginx/sites-available/devops-lab.conf << 'EOF'
# Rate limiting zone: 10 request/detik per IP
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
limit_req_zone $binary_remote_addr zone=app_limit:10m rate=30r/s;

upstream backend_api {
    server 127.0.0.1:8001;
}

upstream frontend_app {
    server 127.0.0.1:8002;
}

server {
    listen 80;
    server_name localhost;

    access_log /var/log/nginx/devops-lab.access.log combined;
    error_log  /var/log/nginx/devops-lab.error.log warn;

    location = / {
        default_type text/html;
        return 200 '<html><body><h1>NGINX Proxy OK</h1></body></html>';
    }

    location /api/ {
        limit_req zone=api_limit burst=20 nodelay;
        proxy_pass http://backend_api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        add_header X-Proxy-By "NGINX";
        add_header X-Upstream "backend-api";
    }

    location /app/ {
        limit_req zone=app_limit burst=50 nodelay;
        proxy_pass http://frontend_app/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        add_header X-Proxy-By "NGINX";
        add_header X-Upstream "frontend-app";
    }

    location /health {
        access_log off;
        return 200 '{"status":"healthy"}\n';
        add_header Content-Type application/json;
    }

    location /nginx-status {
        stub_status on;
        allow 127.0.0.1;
        deny all;
    }
}
EOF

sudo nginx -t && sudo systemctl reload nginx
```

---

## Langkah 8: Test Rate Limiting

```bash
# Kirim banyak request cepat untuk trigger rate limit
for i in {1..50}; do
    curl -s -o /dev/null -w "%{http_code}\n" http://localhost/api/
done

# Request yang melebihi limit akan mendapat 503
```

---

## Langkah 9: Simulasi Backend Down

```bash
# Matikan backend API
kill $(lsof -ti:8001) 2>/dev/null

# Test — NGINX harus return 502 Bad Gateway
curl -sv http://localhost/api/ 2>&1 | grep "HTTP/"
# Output: HTTP/1.1 502 Bad Gateway

# Periksa error log
sudo tail -5 /var/log/nginx/devops-lab.error.log

# Hidupkan kembali backend
cd ~/lab-nginx/api
python3 -m http.server 8001 &

# Verifikasi kembali normal
curl -s http://localhost/api/
```

---

## Langkah 10: Tambah Custom Error Page

```bash
# Buat error pages
sudo mkdir -p /var/www/errors

sudo tee /var/www/errors/50x.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>Service Unavailable</title></head>
<body>
  <h1>503 — Layanan Sedang Tidak Tersedia</h1>
  <p>Backend sedang dalam proses maintenance. Coba lagi dalam beberapa menit.</p>
</body>
</html>
EOF

# Tambahkan ke config NGINX
sudo sed -i '/location \/health/i \    error_page 502 503 504 /50x.html;\n    location = /50x.html {\n        root /var/www/errors;\n        internal;\n    }\n' \
  /etc/nginx/sites-available/devops-lab.conf

sudo nginx -t && sudo systemctl reload nginx
```

---

## Cleanup

```bash
# Matikan backend simulasi
kill $(lsof -ti:8001) 2>/dev/null
kill $(lsof -ti:8002) 2>/dev/null
echo "Backend dihentikan"
```

---

## Verifikasi Akhir

```bash
echo "=== Verifikasi Lab 04 ==="
echo -n "NGINX berjalan: "
  systemctl is-active nginx | grep -q active && echo "✅" || echo "❌"
echo -n "Config syntax OK: "
  sudo nginx -t 2>&1 | grep -q "ok" && echo "✅" || echo "❌"
echo -n "Port 80 listen: "
  ss -tlnp | grep -q ':80 ' && echo "✅" || echo "❌"
echo -n "/health endpoint: "
  curl -sf http://localhost/health | grep -q healthy && echo "✅" || echo "❌"
```

---

## Checkpoint ✅

- [ ] NGINX berhasil terinstall dan berjalan
- [ ] Konfigurasi reverse proxy berhasil dibuat
- [ ] Request ke `/api/` berhasil diteruskan ke backend port 8001
- [ ] Request ke `/app/` berhasil diteruskan ke backend port 8002
- [ ] Header `X-Proxy-By` dan `X-Upstream` muncul di response
- [ ] Rate limiting berhasil dikonfigurasi
- [ ] Memahami error 502 saat backend down
