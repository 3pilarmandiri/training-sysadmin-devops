# LAB 14 — WordPress dengan Docker Compose

## Tujuan
Men-deploy WordPress production-ready menggunakan Docker Compose dengan stack: NGINX + WordPress (PHP-FPM) + MySQL + Redis (object cache), lengkap dengan healthcheck, volume persisten, dan konfigurasi keamanan.

## Prasyarat
- Docker dan Docker Compose terinstall (Lab 01)
- Lab 07 (Docker Compose) selesai — memahami konsep dasar Compose

## Estimasi Waktu
45 – 60 menit

---

## Keunggulan Docker vs LEMP Tradisional

| Aspek | LEMP Tradisional | Docker Compose |
|---|---|---|
| Setup | 60–90 menit | 5 menit |
| Portabilitas | Tergantung OS | Jalan di mana saja |
| Update | Manual per paket | `docker compose pull` |
| Backup | Script custom | Volume snapshot |
| Multi-site | Konfigurasi rumit | Compose file baru |
| Rollback | Sulit | Ganti tag image |

---

## Arsitektur Stack

```
Internet
    │ Port 80 / 443
    ▼
┌─────────────────────────────────────────────┐
│  NGINX (Reverse Proxy + Static Files)        │
│  nginx:1.25-alpine                           │
└────────────────────┬────────────────────────┘
                     │ PHP-FPM :9000
                     ▼
┌─────────────────────────────────────────────┐
│  WordPress (PHP-FPM)                         │
│  wordpress:6.5-php8.2-fpm-alpine            │
│  Terhubung ke MySQL dan Redis               │
└──────────┬──────────────────────┬───────────┘
           │ MySQL :3306          │ Redis :6379
           ▼                      ▼
┌────────────────────┐  ┌──────────────────────┐
│  MySQL 8.0         │  │  Redis 7             │
│  (Database)        │  │  (Object Cache)      │
└────────────────────┘  └──────────────────────┘
           │                      │
    Volume: db_data        Volume: redis_data
           │
    Volume: wp_data (shared WordPress + NGINX)
```

---

## Langkah 1: Siapkan Struktur Proyek

```bash
mkdir -p ~/devops-labs/wordpress-docker
cd ~/devops-labs/wordpress-docker

# Buat semua direktori yang dibutuhkan
mkdir -p nginx/conf.d
mkdir -p nginx/ssl
mkdir -p secrets

echo "Struktur proyek:"
tree . 2>/dev/null || ls -R
```

---

## Langkah 2: Konfigurasi NGINX

```bash
cat > nginx/conf.d/wordpress.conf << 'EOF'
# Upstream ke WordPress PHP-FPM
upstream wordpress_fpm {
    server wordpress:9000;
    keepalive 32;
}

server {
    listen 80;
    server_name localhost wordpress.docker;

    # Root direktori WordPress (volume bersama)
    root /var/www/html;
    index index.php index.html;

    # Log
    access_log /var/log/nginx/wordpress.access.log combined;
    error_log  /var/log/nginx/wordpress.error.log warn;

    # Ukuran upload
    client_max_body_size 64M;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;

    # WordPress permalink
    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    # Proses PHP
    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass wordpress_fpm;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        fastcgi_read_timeout 300;
        fastcgi_buffers 16 16k;
        fastcgi_buffer_size 32k;

        # Real IP ke WordPress
        fastcgi_param HTTP_X_REAL_IP $remote_addr;
        fastcgi_param HTTP_X_FORWARDED_FOR $proxy_add_x_forwarded_for;
    }

    # Cache static assets
    location ~* \.(css|js|jpg|jpeg|png|gif|ico|svg|woff|woff2|ttf|eot|mp4|webp)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
        try_files $uri =404;
    }

    # Blokir akses berbahaya
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }

    location ~* /(?:uploads|files|wp-content)/.*\.php$ {
        deny all;
    }

    # Blokir xmlrpc jika tidak dipakai
    location = /xmlrpc.php {
        deny all;
        access_log off;
    }

    # Batasi akses wp-admin (uncomment untuk production)
    # location /wp-admin {
    #     allow 1.2.3.4;   # IP Anda
    #     deny all;
    # }

    # Health check endpoint
    location = /nginx-health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }

    # NGINX status
    location = /nginx-status {
        stub_status on;
        access_log off;
        allow 127.0.0.1;
        allow 172.16.0.0/12;   # Docker network
        deny all;
    }
}
EOF
```

---

## Langkah 3: Buat File Secret

```bash
# Simpan password di file terpisah (JANGAN commit ke Git!)
echo "WordPressDB@Docker2024" > secrets/db_password.txt
echo "WordPressRoot@Docker2024" > secrets/db_root_password.txt
echo "RedisPass@Docker2024" > secrets/redis_password.txt

# Proteksi permission
chmod 600 secrets/*.txt

# .gitignore untuk secrets
cat > .gitignore << 'EOF'
secrets/
*.env
*.log
.DS_Store
EOF

echo "Secrets terbuat:"
ls -la secrets/
```

---

## Langkah 4: Buat docker-compose.yml

```bash
cat > docker-compose.yml << 'EOF'
services:

  # ─── NGINX — Reverse Proxy ──────────────────────
  nginx:
    image: nginx:1.25-alpine
    container_name: wp-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      # - "443:443"    # Aktifkan setelah setup SSL
    volumes:
      - wp_data:/var/www/html:ro          # WordPress files (read-only untuk NGINX)
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - nginx_logs:/var/log/nginx
      # - ./nginx/ssl:/etc/nginx/ssl:ro   # Aktifkan setelah setup SSL
    depends_on:
      wordpress:
        condition: service_healthy
    networks:
      - frontend
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost/nginx-health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s

  # ─── WordPress — PHP-FPM ────────────────────────
  wordpress:
    image: wordpress:6.5-php8.2-fpm-alpine
    container_name: wp-app
    restart: unless-stopped
    expose:
      - "9000"
    volumes:
      - wp_data:/var/www/html              # WordPress files (read-write)
      - ./wp-custom.ini:/usr/local/etc/php/conf.d/custom.ini:ro
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_NAME: wordpress_db
      WORDPRESS_DB_USER: wp_user
      WORDPRESS_DB_PASSWORD_FILE: /run/secrets/db_password
      WORDPRESS_TABLE_PREFIX: wp_dc_
      WORDPRESS_DEBUG: "false"
      # WordPress config extras
      WORDPRESS_CONFIG_EXTRA: |
        define('WP_REDIS_HOST', 'cache');
        define('WP_REDIS_PORT', 6379);
        define('WP_REDIS_PASSWORD', 'RedisPass@Docker2024');
        define('WP_CACHE', true);
        define('WP_POST_REVISIONS', 5);
        define('DISALLOW_FILE_EDIT', true);
        define('WP_DEBUG', false);
        define('WP_MEMORY_LIMIT', '256M');
        define('WP_MAX_MEMORY_LIMIT', '512M');
        @ini_set('upload_max_size', '64M');
        @ini_set('post_max_size', '64M');
        @ini_set('upload_max_filesize', '64M');
        define('FS_METHOD', 'direct');
    secrets:
      - db_password
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_healthy
    networks:
      - frontend
      - backend
    healthcheck:
      test: ["CMD", "php", "-r", "echo 'ok';"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 30s

  # ─── MySQL — Database ───────────────────────────
  db:
    image: mysql:8.0
    container_name: wp-db
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: wordpress_db
      MYSQL_USER: wp_user
      MYSQL_PASSWORD_FILE: /run/secrets/db_password
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_root_password
    secrets:
      - db_password
      - db_root_password
    volumes:
      - db_data:/var/lib/mysql
      - ./mysql-tuning.cnf:/etc/mysql/conf.d/tuning.cnf:ro
    networks:
      - backend
    command: >
      --default-authentication-plugin=mysql_native_password
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_unicode_ci
      --innodb-buffer-pool-size=256M
      --max-connections=100
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost",
             "-u", "wp_user", "--password=WordPressDB@Docker2024"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  # ─── Redis — Object Cache ───────────────────────
  cache:
    image: redis:7-alpine
    container_name: wp-cache
    restart: unless-stopped
    command: >
      redis-server
      --requirepass RedisPass@Docker2024
      --maxmemory 128mb
      --maxmemory-policy allkeys-lru
      --save 60 1
      --loglevel warning
    volumes:
      - redis_data:/data
    networks:
      - backend
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "RedisPass@Docker2024", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  wp_data:
    driver: local
  db_data:
    driver: local
  redis_data:
    driver: local
  nginx_logs:
    driver: local

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true        # MySQL dan Redis tidak terekspos ke host

secrets:
  db_password:
    file: ./secrets/db_password.txt
  db_root_password:
    file: ./secrets/db_root_password.txt
EOF
```

---

## Langkah 5: Buat File Konfigurasi Tambahan

```bash
# PHP custom settings untuk WordPress
cat > wp-custom.ini << 'EOF'
; WordPress PHP Configuration
upload_max_filesize = 64M
post_max_size = 64M
memory_limit = 256M
max_execution_time = 300
max_input_time = 300
max_input_vars = 3000
date.timezone = Asia/Jakarta

; Opcache (performa)
opcache.enable = 1
opcache.memory_consumption = 128
opcache.interned_strings_buffer = 8
opcache.max_accelerated_files = 10000
opcache.revalidate_freq = 60
opcache.fast_shutdown = 1

; Keamanan
expose_php = Off
allow_url_fopen = On
EOF

# MySQL tuning untuk server kecil
cat > mysql-tuning.cnf << 'EOF'
[mysqld]
# Encoding
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

# Performance
innodb_buffer_pool_size = 256M
innodb_log_file_size = 64M
innodb_flush_method = O_DIRECT
innodb_file_per_table = 1

# Connection
max_connections = 100
wait_timeout = 600
interactive_timeout = 600

# Query cache (MySQL 8 tidak support query_cache, skip)
# Slow query log
slow_query_log = 1
slow_query_log_file = /var/lib/mysql/slow.log
long_query_time = 2
EOF
```

---

## Langkah 6: Jalankan Stack

```bash
# Build dan start semua service
docker compose up -d

# Monitor startup progress
echo "Menunggu semua service healthy (sekitar 60 detik)..."
sleep 10

# Pantau status
watch -n 3 "docker compose ps" 
# Tekan Ctrl+C setelah semua healthy

# Atau pantau tanpa watch:
for i in {1..12}; do
    echo "=== Status (attempt $i) ==="
    docker compose ps
    echo ""
    sleep 5
done
```

Output yang diharapkan:
```
NAME         STATUS
wp-app       Up (healthy)
wp-cache     Up (healthy)
wp-db        Up (healthy)
wp-nginx     Up (healthy)
```

---

## Langkah 7: Akses WordPress

```bash
# Tambahkan ke /etc/hosts
echo "127.0.0.1 wordpress.docker" | sudo tee -a /etc/hosts

# Test akses
curl -I http://wordpress.docker
curl -I http://localhost
```

Buka browser dan akses:
```
http://localhost
atau
http://wordpress.docker
```

Ikuti wizard instalasi WordPress:

| Field | Nilai |
|---|---|
| Judul Situs | `DevOps Training Blog` |
| Nama pengguna | `devops_admin` |
| Kata sandi | Minimal 12 karakter, gunakan password manager |
| Email | `admin@example.com` |

---

## Langkah 8: Install Plugin Redis Object Cache

Setelah WordPress terinstall:

1. Login ke **wp-admin** → **Plugin** → **Tambah Baru**
2. Cari: `Redis Object Cache`
3. Install dan Aktifkan
4. Buka **Pengaturan** → **Redis** → klik **Enable Object Cache**

Atau via CLI (WP-CLI di dalam container):

```bash
# Masuk ke container WordPress
docker compose exec wordpress sh

# Install WP-CLI
curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
chmod +x wp-cli.phar
mv wp-cli.phar /usr/local/bin/wp

# Install plugin Redis Object Cache
wp plugin install redis-cache --activate --allow-root
wp redis enable --allow-root

# Verifikasi
wp redis status --allow-root

exit
```

---

## Langkah 9: Verifikasi Stack Lengkap

```bash
echo "=== Verifikasi Stack WordPress Docker ==="

echo -n "NGINX berjalan: "
  docker compose ps nginx | grep -q "healthy" && echo "✅" || echo "⏳"

echo -n "WordPress berjalan: "
  docker compose ps wordpress | grep -q "healthy" && echo "✅" || echo "⏳"

echo -n "MySQL berjalan: "
  docker compose ps db | grep -q "healthy" && echo "✅" || echo "⏳"

echo -n "Redis berjalan: "
  docker compose ps cache | grep -q "healthy" && echo "✅" || echo "⏳"

echo -n "Homepage accessible: "
  curl -sf http://localhost | grep -qi "wordpress\|DOCTYPE" && echo "✅" || echo "❌"

echo -n "wp-config.php terbuat: "
  docker compose exec wordpress test -f /var/www/html/wp-config.php && echo "✅" || echo "⏳ (belum install)"

echo -n "Redis terhubung: "
  docker compose exec cache redis-cli -a RedisPass@Docker2024 ping | grep -q PONG && echo "✅" || echo "❌"

echo -n "MySQL database ada: "
  docker compose exec db mysql -u wp_user -pWordPressDB@Docker2024 \
    -e "SHOW DATABASES;" 2>/dev/null | grep -q wordpress_db && echo "✅" || echo "❌"

echo ""
echo "=== Resource Usage ==="
docker stats --no-stream $(docker compose ps -q)
```

---

## Langkah 10: Backup dan Restore

```bash
# ─── BACKUP ─────────────────────────────────────
cat > backup-wp-docker.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="$HOME/wp-docker-backups"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p "$BACKUP_DIR"
cd ~/devops-labs/wordpress-docker

echo "=== Backup WordPress Docker Stack - $DATE ==="

# 1. Backup database
echo "Backup database..."
docker compose exec -T db mysqldump \
  -u wp_user -pWordPressDB@Docker2024 wordpress_db | \
  gzip > "$BACKUP_DIR/db_${DATE}.sql.gz"
echo "  ✅ Database backup: db_${DATE}.sql.gz"

# 2. Backup WordPress files (dari volume)
echo "Backup WordPress files..."
docker run --rm \
  --volumes-from wp-app \
  -v "$BACKUP_DIR":/backup \
  alpine:latest \
  tar czf /backup/wp_files_${DATE}.tar.gz \
  --exclude=/var/www/html/wp-content/cache \
  /var/www/html
echo "  ✅ Files backup: wp_files_${DATE}.tar.gz"

# 3. Backup config files
tar czf "$BACKUP_DIR/config_${DATE}.tar.gz" \
  nginx/ mysql-tuning.cnf wp-custom.ini docker-compose.yml
echo "  ✅ Config backup: config_${DATE}.tar.gz"

# 4. Hapus backup > 7 hari
find "$BACKUP_DIR" -mtime +7 -delete

echo ""
echo "Backup selesai!"
ls -lh "$BACKUP_DIR"
EOF

chmod +x backup-wp-docker.sh
echo "Script backup siap: ./backup-wp-docker.sh"
```

---

## Langkah 11: Update WordPress

```bash
# Update image ke versi terbaru
docker compose pull wordpress mysql redis nginx

# Restart service dengan image baru (tanpa downtime di WordPress)
docker compose up -d --no-deps --build wordpress

# Jika update major (mis: WordPress 6.5 → 7.0):
# 1. Backup dulu!
# 2. docker compose pull wordpress
# 3. docker compose up -d --no-deps wordpress
# 4. Verifikasi di admin → Dashboard → Update
```

---

## Langkah 12: Monitoring Sederhana

```bash
# Lihat log semua service
docker compose logs -f --tail=50

# Log service tertentu
docker compose logs -f nginx
docker compose logs -f wordpress

# Pantau request WordPress
docker compose exec nginx \
  tail -f /var/log/nginx/wordpress.access.log | \
  awk '{print $1, $7, $9}'    # IP, URL, Status code

# Cek penggunaan resource
docker stats $(docker compose ps -q) --format \
  "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"
```

---

## Cleanup

```bash
# Stop semua container
docker compose stop

# Hapus container (data volume tetap ada)
docker compose down

# Hapus juga semua volume (DATA HILANG!)
# docker compose down -v

echo "WordPress Docker stack dihentikan"
```

---

## Checkpoint ✅

- [ ] Stack 4 service berjalan (NGINX, WordPress, MySQL, Redis)
- [ ] Semua service berstatus `healthy`
- [ ] WordPress wizard berhasil diselesaikan via browser
- [ ] Redis Object Cache terinstall dan aktif
- [ ] Backup script berhasil dijalankan
- [ ] Memahami perbedaan volume `wp_data`, `db_data`, dan `redis_data`
- [ ] NGINX berhasil meneruskan request PHP ke WordPress container
- [ ] Backend network (MySQL + Redis) tidak terekspos ke host
