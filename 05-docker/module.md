# BAB 5 — Docker & Containerization

## Tujuan Pembelajaran

Setelah menyelesaikan bab ini, peserta mampu:
- Memahami perbedaan container vs virtual machine
- Menulis Dockerfile yang efisien dan aman
- Mengelola Docker image dan container
- Menggunakan Docker Compose untuk multi-container application
- Menerapkan best practice Docker untuk production

---

## 5.1 Konsep Container

### VM vs Container

```
VIRTUAL MACHINE                    CONTAINER
┌─────────────────────────┐        ┌─────────────────────────┐
│  App A  │  App B         │        │  App A  │  App B         │
├─────────┼────────────────┤        ├─────────┼────────────────┤
│ Guest OS│ Guest OS       │        │         │                │
│(Ubuntu) │ (CentOS)       │        │  Container Runtime       │
├─────────┴────────────────┤        │  (Docker Engine)         │
│     Hypervisor           │        ├──────────────────────────┤
├──────────────────────────┤        │     Host OS (Linux)      │
│     Host OS              │        ├──────────────────────────┤
├──────────────────────────┤        │     Hardware             │
│     Hardware             │        └──────────────────────────┘
└──────────────────────────┘
Overhead besar, boot lambat        Lightweight, boot cepat
Size: GiB                          Size: MiB
Isolasi penuh                      Isolasi proses (namespace)
```

### Keunggulan Container

| Aspek | Keunggulan |
|---|---|
| **Portabilitas** | "Build once, run anywhere" — sama di laptop, CI, dan production |
| **Konsistensi** | Environment identik di semua tahap (dev, staging, prod) |
| **Isolasi** | Setiap app berjalan di namespace terpisah |
| **Resource** | Lebih ringan dari VM, bisa jalan ratusan container di satu host |
| **Kecepatan** | Start dalam hitungan detik, bukan menit |
| **Skalabilitas** | Mudah di-scale horizontal |

---

## 5.2 Arsitektur Docker

```
Docker CLI          Docker Daemon (dockerd)         Registry
    │                      │                    (Docker Hub, ECR, GCR)
    │──── docker build ────▶│                           │
    │──── docker run ───────▶│                           │
    │                      │──── pull image ───────────▶│
    │                      │◀─── image layers ──────────│
    │                      │
    │              ┌────────┴────────┐
    │              │  Container      │
    │              │  (running image)│
    │              └────────┬────────┘
    │                      │
    │              ┌────────┴────────┐
    │              │ Image Layers    │
    │              │ (read-only)     │
    │              └─────────────────┘
```

### Komponen Utama

| Komponen | Penjelasan |
|---|---|
| **Image** | Template read-only berisi OS + app + dependencies |
| **Container** | Instance yang berjalan dari sebuah image |
| **Layer** | Image terdiri dari lapisan-lapisan; setiap instruksi Dockerfile = 1 layer |
| **Registry** | Repository untuk menyimpan dan mendistribusikan image |
| **Volume** | Storage persisten yang tidak ikut terhapus saat container mati |
| **Network** | Jaringan virtual yang menghubungkan container |

---

## 5.3 Perintah Docker Dasar

### Manajemen Image

```bash
# Pull image dari registry
docker pull nginx
docker pull nginx:1.25-alpine    # Versi spesifik
docker pull python:3.12-slim

# List image
docker images
docker image ls
docker image ls --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# Hapus image
docker rmi nginx
docker image rm nginx:1.25-alpine
docker image prune               # Hapus image yang tidak terpakai

# Inspeksi image
docker inspect nginx
docker history nginx             # Lihat layer history
```

### Manajemen Container

```bash
# Jalankan container
docker run nginx                             # Foreground
docker run -d nginx                          # Detached (background)
docker run -d -p 8080:80 nginx               # Port mapping: host:container
docker run -d --name webserver nginx         # Beri nama
docker run -it ubuntu bash                   # Interactive + TTY
docker run --rm ubuntu echo "hello"          # Auto-hapus setelah selesai

# List container
docker ps                        # Container berjalan
docker ps -a                     # Semua container (termasuk stopped)

# Start, stop, restart
docker stop webserver
docker start webserver
docker restart webserver
docker kill webserver            # SIGKILL (paksa)

# Masuk ke container yang berjalan
docker exec -it webserver bash
docker exec webserver cat /etc/nginx/nginx.conf

# Lihat log
docker logs webserver
docker logs -f webserver         # Follow log
docker logs --tail 50 webserver  # 50 baris terakhir

# Hapus container
docker rm webserver
docker rm -f webserver           # Paksa (meskipun masih berjalan)
docker container prune           # Hapus semua stopped container

# Statistik resource
docker stats
docker stats webserver
```

---

## 5.4 Dockerfile — Membangun Image

### Instruksi Dockerfile

```dockerfile
# Instruksi lengkap dengan penjelasan

# FROM — base image (wajib, harus pertama)
FROM python:3.12-slim

# ARG — variabel saat build (tidak tersedia saat runtime)
ARG APP_VERSION=1.0.0

# ENV — variabel environment (tersedia saat runtime)
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    APP_VERSION=${APP_VERSION}

# LABEL — metadata image
LABEL maintainer="team@example.com" \
      version="${APP_VERSION}" \
      description="DevOps Training App"

# WORKDIR — set direktori kerja (buat jika belum ada)
WORKDIR /app

# RUN — jalankan perintah saat build (buat layer baru)
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

# COPY — salin file dari host ke image
# (lebih direkomendasikan dari ADD untuk file lokal)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# RUN untuk setup user non-root (keamanan)
RUN groupadd -r appuser && useradd -r -g appuser appuser
RUN chown -R appuser:appuser /app
USER appuser

# EXPOSE — dokumentasi port yang digunakan (tidak publish otomatis)
EXPOSE 8080

# HEALTHCHECK — cek kesehatan container
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# VOLUME — direktori yang dijadikan mount point
VOLUME ["/app/data", "/app/logs"]

# ENTRYPOINT — perintah utama yang selalu dijalankan
ENTRYPOINT ["python3"]

# CMD — argumen default untuk ENTRYPOINT (bisa di-override)
CMD ["app.py"]
```

### Multi-Stage Build (Best Practice)

Multi-stage build menghasilkan image production yang kecil:

```dockerfile
# Stage 1: Builder (hanya untuk build, tidak masuk image final)
FROM python:3.12 AS builder

WORKDIR /build
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Stage 2: Production image (kecil dan bersih)
FROM python:3.12-slim AS production

WORKDIR /app

# Salin hanya hasil install dari builder
COPY --from=builder /root/.local /root/.local

COPY src/ ./src/
COPY config/ ./config/

ENV PATH=/root/.local/bin:$PATH \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

RUN groupadd -r app && useradd -r -g app app
USER app

EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s \
    CMD curl -f http://localhost:8080/health || exit 1

ENTRYPOINT ["python3", "src/app.py"]
```

---

## 5.5 Docker Volume dan Network

### Volume

```bash
# Buat volume
docker volume create mydata

# List volume
docker volume ls

# Gunakan volume saat run
docker run -d \
  --name db \
  -v mydata:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  mysql:8.0

# Bind mount (map direktori host ke container)
docker run -d \
  -v $(pwd)/html:/usr/share/nginx/html:ro \
  -p 8080:80 \
  nginx

# Inspeksi volume
docker volume inspect mydata

# Hapus volume
docker volume rm mydata
docker volume prune            # Hapus semua unused volume
```

### Network

```bash
# List network
docker network ls

# Network types:
# bridge  — default, container terisolasi dalam satu host
# host    — container berbagi network host (no isolation)
# none    — tanpa network
# overlay — lintas host (Swarm/Kubernetes)

# Buat network custom
docker network create mynet
docker network create \
  --driver bridge \
  --subnet 172.20.0.0/16 \
  --gateway 172.20.0.1 \
  devops-net

# Jalankan container dalam network
docker run -d --name app1 --network mynet nginx
docker run -d --name app2 --network mynet nginx

# Container dalam network yang sama bisa saling ping by name
docker exec app1 ping app2

# Hubungkan container ke network
docker network connect mynet existing-container

# Inspeksi network
docker network inspect mynet
```

---

## 5.6 Docker Compose

Docker Compose mengelola aplikasi multi-container menggunakan file YAML.

### Struktur File docker-compose.yml

```yaml
version: "3.9"         # Versi schema Compose

services:
  # Definisi setiap container
  web:
    image: nginx:1.25-alpine
    build:               # Atau gunakan build context
      context: .
      dockerfile: Dockerfile
      args:
        APP_VERSION: "1.0.0"
    container_name: devops-web
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - static_data:/var/www/html
    environment:
      - NGINX_HOST=localhost
    env_file:
      - .env
    depends_on:
      app:
        condition: service_healthy
    networks:
      - frontend
    labels:
      - "traefik.enable=true"

  app:
    build: ./app
    container_name: devops-app
    restart: unless-stopped
    expose:
      - "8080"
    environment:
      DATABASE_URL: postgresql://user:pass@db:5432/mydb
      REDIS_URL: redis://cache:6379
    volumes:
      - ./app:/code:delegated
      - app_logs:/app/logs
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    depends_on:
      - db
      - cache
    networks:
      - frontend
      - backend

  db:
    image: postgres:16-alpine
    container_name: devops-db
    restart: unless-stopped
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: user
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    volumes:
      - pg_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    secrets:
      - db_password
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5

  cache:
    image: redis:7-alpine
    container_name: devops-cache
    restart: unless-stopped
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    networks:
      - backend

volumes:
  pg_data:
  redis_data:
  static_data:
  app_logs:

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true          # Tidak terekspos ke host

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

### Perintah Docker Compose

```bash
# Start semua service (detached)
docker compose up -d

# Start dan build ulang image
docker compose up -d --build

# Lihat status service
docker compose ps
docker compose top

# Lihat log
docker compose logs -f
docker compose logs -f app db     # Service tertentu

# Stop semua service
docker compose stop

# Stop + hapus container (volume tetap ada)
docker compose down

# Stop + hapus container + volume
docker compose down -v

# Scale service
docker compose up -d --scale app=3

# Jalankan perintah di service
docker compose exec app bash
docker compose exec db psql -U user -d mydb

# Restart service tertentu
docker compose restart app

# Pull image terbaru
docker compose pull
```

---

## 5.7 Docker Best Practices

### 1. Gunakan Image Base yang Tepat

```dockerfile
# Hindari — terlalu besar
FROM ubuntu:22.04

# Lebih baik — sudah ada runtime
FROM python:3.12

# Terbaik — minimal (slim/alpine)
FROM python:3.12-slim
FROM python:3.12-alpine  # Lebih kecil tapi ada keterbatasan
```

### 2. Susun Layer dengan Cermat

```dockerfile
# Buruk — setiap perubahan app menyebabkan reinstall dependencies
COPY . .
RUN pip install -r requirements.txt

# Baik — cache dependencies jika requirements.txt tidak berubah
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

### 3. Minimalkan Jumlah Layer RUN

```dockerfile
# Buruk — 3 layer
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# Baik — 1 layer
RUN apt-get update && apt-get install -y \
    curl \
    && rm -rf /var/lib/apt/lists/*
```

### 4. Jangan Simpan Secret di Image

```bash
# JANGAN ini — secret tersimpan di layer image
ENV DB_PASSWORD=secret123

# Gunakan env file saat runtime
docker run --env-file .env myapp

# Atau Docker secrets (Swarm)
# Atau Kubernetes Secrets
```

### 5. Gunakan .dockerignore

```dockerignore
# .dockerignore
.git
.gitignore
node_modules
__pycache__
*.pyc
.env
*.log
tests/
docs/
README.md
Dockerfile
docker-compose*.yml
```

---

## Ringkasan

| Konsep | Perintah |
|---|---|
| Image | `docker pull`, `docker build`, `docker push`, `docker images` |
| Container | `docker run`, `docker exec`, `docker logs`, `docker stop/rm` |
| Volume | `docker volume create/ls/rm`, `-v host:container` |
| Network | `docker network create/ls/connect` |
| Compose | `docker compose up/down/logs/exec/scale` |
| Build | `docker build -t name:tag .`, `--no-cache`, `--build-arg` |
