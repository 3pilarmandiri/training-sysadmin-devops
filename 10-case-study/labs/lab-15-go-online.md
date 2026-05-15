# LAB 15 — Go Online: Domain, NGINX, dan HTTPS

## Tujuan
Membuat WordPress yang sudah terinstall (Lab 13 atau Lab 14) bisa diakses dari internet dengan domain custom, konfigurasi NGINX production-ready, dan sertifikat SSL/HTTPS gratis menggunakan Let's Encrypt + Certbot.

## Prasyarat
- Lab 13 (LEMP Stack) atau Lab 14 (Docker) selesai — WordPress sudah jalan di server
- **VPS dengan IP Publik** (DigitalOcean, IDCloudHost, Biznet Gio, AWS EC2, dll)
- **Domain yang sudah terdaftar** (Niagahoster, Namecheap, Cloudflare, dll)
- Port 80 dan 443 terbuka di firewall VPS

## Estimasi Waktu
45 – 60 menit

---

## Gambaran Alur "Go Online"

```
┌──────────────────────────────────────────────────────────┐
│                    ALUR GO ONLINE                        │
└──────────────────────────────────────────────────────────┘

LANGKAH 1: Daftarkan domain
  contoh.com  →  dibeli di Niagahoster/Namecheap

LANGKAH 2: Arahkan DNS ke IP VPS
  contoh.com       A    → 1.2.3.4   (IP VPS Anda)
  www.contoh.com   CNAME → contoh.com

LANGKAH 3: Tunggu propagasi DNS (5 menit – 48 jam)
  nslookup contoh.com  → harus resolve ke 1.2.3.4

LANGKAH 4: Install Certbot + request sertifikat SSL
  certbot → verifikasi domain ownership (via port 80)
  → terbitkan cert di /etc/letsencrypt/live/contoh.com/

LANGKAH 5: Konfigurasi NGINX dengan HTTPS
  listen 443 ssl;
  ssl_certificate /etc/letsencrypt/live/contoh.com/fullchain.pem;

LANGKAH 6: Verifikasi
  https://contoh.com  →  ✅ WordPress online dengan HTTPS
```

---

## TAHAP 1: Persiapan VPS

### Langkah 1.1: Setup Awal VPS

```bash
# Update sistem
sudo apt update && sudo apt upgrade -y

# Install tools dasar (jika belum dari Lab 01)
sudo apt install -y curl wget git vim ufw fail2ban

# Set hostname
sudo hostnamectl set-hostname nama-server-anda

# Verifikasi IP publik
curl -s ifconfig.me
curl -s https://api.ipify.org
echo ""
```

### Langkah 1.2: Konfigurasi Firewall

```bash
# Konfigurasi UFW
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH (JANGAN lupa ini atau Anda terkunci keluar!)
sudo ufw allow 22/tcp

# Allow HTTP dan HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Enable firewall
sudo ufw enable

# Verifikasi
sudo ufw status verbose
```

### Langkah 1.3: Basic Security Hardening

```bash
# Konfigurasi fail2ban (proteksi brute force SSH)
sudo systemctl enable --now fail2ban

# Konfigurasi SSH lebih ketat
sudo tee /etc/ssh/sshd_config.d/hardening.conf << 'EOF'
# Matikan root login
PermitRootLogin no

# Matikan password auth (gunakan SSH key)
# PasswordAuthentication no    # Aktifkan jika sudah setup SSH key

# Batasi percobaan auth
MaxAuthTries 3
MaxSessions 3

# Timeout
ClientAliveInterval 300
ClientAliveCountMax 2
EOF

sudo systemctl restart sshd
```

---

## TAHAP 2: Setup DNS

### Langkah 2.1: Temukan IP Publik VPS

```bash
# Catat IP publik Anda
MY_IP=$(curl -s https://api.ipify.org)
echo "IP Publik VPS: $MY_IP"
```

### Langkah 2.2: Konfigurasi DNS Record

Login ke control panel domain Anda (Niagahoster, Cloudflare, Namecheap, dll) dan tambahkan DNS record:

| Tipe | Nama/Host | Nilai | TTL |
|---|---|---|---|
| `A` | `@` (root domain) | `IP_PUBLIK_VPS` | 300 |
| `A` | `www` | `IP_PUBLIK_VPS` | 300 |
| `CNAME` | `www` | `contoh.com` | 300 |

**Contoh untuk domain `blog.contoh.com`:**

| Tipe | Nama | Nilai |
|---|---|---|
| `A` | `blog` | `1.2.3.4` |

### Langkah 2.3: Verifikasi Propagasi DNS

```bash
DOMAIN="contoh.com"    # Ganti dengan domain Anda

# Cek resolusi DNS
nslookup $DOMAIN
dig $DOMAIN A +short

# Cek dari berbagai DNS server
dig @8.8.8.8 $DOMAIN A +short        # Google DNS
dig @1.1.1.1 $DOMAIN A +short        # Cloudflare DNS
dig @208.67.222.222 $DOMAIN A +short  # OpenDNS

# Online checker: https://www.whatsmydns.net/
echo ""
echo "Jika hasilnya adalah IP VPS Anda ($MY_IP), propagasi berhasil!"
echo "Jika belum, tunggu 5-60 menit dan coba lagi."
```

> **Catatan Cloudflare:** Jika menggunakan Cloudflare sebagai DNS, matikan dulu proxy (ikon awan oranye → ikon awan abu-abu) saat setup SSL. Aktifkan kembali setelah berhasil.

---

## TAHAP 3: Install Certbot dan SSL

### Langkah 3.1: Install Certbot

```bash
# Install Certbot via snap (cara resmi terbaru)
sudo apt install -y snapd
sudo snap install core
sudo snap refresh core
sudo snap install --classic certbot

# Buat symlink
sudo ln -sf /snap/bin/certbot /usr/bin/certbot

# Verifikasi
certbot --version
```

### Langkah 3.2: Request Sertifikat SSL

#### Untuk Lab 13 (LEMP Stack — NGINX sudah berjalan):

```bash
DOMAIN="contoh.com"    # Ganti dengan domain Anda

# Request certificate + auto-konfigurasi NGINX
sudo certbot --nginx \
  -d $DOMAIN \
  -d www.$DOMAIN \
  --non-interactive \
  --agree-tos \
  --email admin@$DOMAIN \
  --redirect    # Otomatis redirect HTTP → HTTPS

# Jika berhasil, output-nya:
# Successfully received certificate.
# Certificate is saved at: /etc/letsencrypt/live/contoh.com/fullchain.pem
# Key is saved at: /etc/letsencrypt/live/contoh.com/privkey.pem
```

#### Untuk Lab 14 (Docker Compose — NGINX di container):

Gunakan **standalone mode** karena NGINX ada di container:

```bash
DOMAIN="contoh.com"    # Ganti dengan domain Anda

# Stop NGINX container sementara (agar port 80 bebas)
cd ~/devops-labs/wordpress-docker
docker compose stop nginx

# Request certificate standalone
sudo certbot certonly --standalone \
  -d $DOMAIN \
  -d www.$DOMAIN \
  --non-interactive \
  --agree-tos \
  --email admin@$DOMAIN

# Salin sertifikat ke direktori proyek
sudo mkdir -p nginx/ssl
sudo cp /etc/letsencrypt/live/$DOMAIN/fullchain.pem nginx/ssl/
sudo cp /etc/letsencrypt/live/$DOMAIN/privkey.pem nginx/ssl/
sudo chown $USER:$USER nginx/ssl/*.pem

# Start NGINX kembali (setelah update config HTTPS)
```

### Langkah 3.3: Verifikasi Sertifikat

```bash
# List sertifikat yang ada
sudo certbot certificates

# Cek expiry
sudo certbot certificates | grep -A3 $DOMAIN

# Test renewal
sudo certbot renew --dry-run
```

---

## TAHAP 4: Konfigurasi NGINX Production dengan HTTPS

### Untuk Lab 13 (LEMP Stack):

Certbot sudah otomatis menambahkan blok SSL. Perlu sedikit modifikasi:

```bash
DOMAIN="contoh.com"

sudo tee /etc/nginx/sites-available/wordpress.conf << NGINXEOF
# Redirect HTTP → HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name ${DOMAIN} www.${DOMAIN};

    # Let's Encrypt challenge
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
        allow all;
    }

    # Redirect semua traffic ke HTTPS
    location / {
        return 301 https://\$host\$request_uri;
    }
}

# HTTPS Server
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name ${DOMAIN} www.${DOMAIN};

    # SSL Certificate dari Let's Encrypt
    ssl_certificate /etc/letsencrypt/live/${DOMAIN}/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/${DOMAIN}/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/${DOMAIN}/chain.pem;

    # SSL Hardening (Mozilla Intermediate config)
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 1.1.1.1 valid=300s;
    resolver_timeout 5s;

    # HSTS (aktifkan setelah yakin HTTPS berjalan)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    # Security Headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;

    # WordPress root
    root /var/www/wordpress;
    index index.php index.html;

    # Log
    access_log /var/log/nginx/wordpress.access.log combined;
    error_log  /var/log/nginx/wordpress.error.log warn;

    client_max_body_size 64M;

    # WordPress permalink
    location / {
        try_files \$uri \$uri/ /index.php?\$args;
    }

    # PHP-FPM
    location ~ \.php\$ {
        include fastcgi_params;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME \$document_root\$fastcgi_script_name;
        fastcgi_read_timeout 300;
        fastcgi_buffers 16 16k;
        fastcgi_buffer_size 32k;
    }

    # Cache static
    location ~* \.(css|js|jpg|jpeg|png|gif|ico|svg|woff|woff2)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # Blokir akses berbahaya
    location ~ /\. { deny all; }
    location ~* /(?:uploads|files)/.*\.php\$ { deny all; }
    location = /xmlrpc.php { deny all; }
}
NGINXEOF

sudo nginx -t && sudo systemctl reload nginx
echo "NGINX dikonfigurasi ulang dengan HTTPS"
```

### Untuk Lab 14 (Docker Compose):

Update konfigurasi NGINX container untuk HTTPS:

```bash
cd ~/devops-labs/wordpress-docker
DOMAIN="contoh.com"

cat > nginx/conf.d/wordpress.conf << NGINXEOF
upstream wordpress_fpm {
    server wordpress:9000;
}

# HTTP → HTTPS redirect
server {
    listen 80;
    server_name ${DOMAIN} www.${DOMAIN};

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://\$host\$request_uri;
    }
}

# HTTPS
server {
    listen 443 ssl http2;
    server_name ${DOMAIN} www.${DOMAIN};

    ssl_certificate /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_stapling on;
    ssl_stapling_verify on;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;

    root /var/www/html;
    index index.php;

    client_max_body_size 64M;

    location / {
        try_files \$uri \$uri/ /index.php?\$args;
    }

    location ~ \.php\$ {
        fastcgi_pass wordpress_fpm;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME \$document_root\$fastcgi_script_name;
        fastcgi_read_timeout 300;
    }

    location ~* \.(css|js|jpg|jpeg|png|gif|ico|svg|woff|woff2)\$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    location ~ /\. { deny all; }
    location ~* /(?:uploads|files)/.*\.php\$ { deny all; }
    location = /xmlrpc.php { deny all; }
}
NGINXEOF

# Update docker-compose.yml untuk mount SSL dan buka port 443
# (edit bagian ports nginx):
# ports:
#   - "80:80"
#   - "443:443"
# volumes tambahkan:
#   - ./nginx/ssl:/etc/nginx/ssl:ro

docker compose restart nginx
```

---

## TAHAP 5: Update WordPress URL ke HTTPS

### Langkah 5.1: Update URL di WordPress

Login ke **wp-admin** → **Pengaturan** → **Umum**:

```
Alamat WordPress (URL): https://contoh.com
Alamat Situs (URL):     https://contoh.com
```

Klik **Simpan Perubahan**.

### Langkah 5.2: Update via WP-CLI (Alternatif)

```bash
# Untuk LEMP Stack:
sudo -u www-data wp --path=/var/www/wordpress option update siteurl "https://contoh.com"
sudo -u www-data wp --path=/var/www/wordpress option update home "https://contoh.com"

# Untuk Docker:
docker compose exec wordpress wp --allow-root option update siteurl "https://contoh.com"
docker compose exec wordpress wp --allow-root option update home "https://contoh.com"
```

### Langkah 5.3: Search-Replace URL Lama di Database

```bash
# Untuk LEMP Stack:
sudo -u www-data wp --path=/var/www/wordpress search-replace \
  'http://contoh.com' 'https://contoh.com' \
  --all-tables --report-changed-only

# Untuk Docker:
docker compose exec wordpress wp --allow-root search-replace \
  'http://contoh.com' 'https://contoh.com' \
  --all-tables --report-changed-only
```

---

## TAHAP 6: Auto-Renewal SSL

Let's Encrypt menerbitkan sertifikat yang berlaku **90 hari**. Certbot otomatis memperbarui, tapi perlu pastikan renewal bekerja.

```bash
# Cek apakah timer renewal sudah aktif
sudo systemctl list-timers | grep certbot

# Atau cek crontab
sudo crontab -l

# Test renewal (dry-run)
sudo certbot renew --dry-run

# Jika menggunakan Docker, perlu hook setelah renewal
# agar NGINX container me-reload sertifikat baru:
sudo mkdir -p /etc/letsencrypt/renewal-hooks/post

# Buat hook untuk Docker setup:
sudo tee /etc/letsencrypt/renewal-hooks/post/reload-nginx.sh << 'EOF'
#!/bin/bash
# Salin cert baru ke direktori proyek
cp /etc/letsencrypt/live/contoh.com/fullchain.pem \
   /home/ubuntu/devops-labs/wordpress-docker/nginx/ssl/fullchain.pem
cp /etc/letsencrypt/live/contoh.com/privkey.pem \
   /home/ubuntu/devops-labs/wordpress-docker/nginx/ssl/privkey.pem

# Reload NGINX container
cd /home/ubuntu/devops-labs/wordpress-docker
docker compose exec nginx nginx -s reload

echo "SSL cert diperbarui dan NGINX direload"
EOF

sudo chmod +x /etc/letsencrypt/renewal-hooks/post/reload-nginx.sh

# Buat hook untuk LEMP Stack:
sudo tee /etc/letsencrypt/renewal-hooks/post/reload-lemp-nginx.sh << 'EOF'
#!/bin/bash
systemctl reload nginx
echo "NGINX direload setelah renewal SSL"
EOF

sudo chmod +x /etc/letsencrypt/renewal-hooks/post/reload-lemp-nginx.sh

# Jadwalkan di crontab (jika timer belum ada)
(crontab -l 2>/dev/null; echo "0 3 * * * certbot renew --quiet") | crontab -
```

---

## TAHAP 7: Verifikasi Go Online

```bash
DOMAIN="contoh.com"   # Ganti dengan domain Anda

echo "=== Verifikasi Go Online ==="

echo -n "DNS resolve ke IP yang benar: "
  RESOLVED=$(dig +short $DOMAIN)
  MY_IP=$(curl -s ifconfig.me)
  [ "$RESOLVED" = "$MY_IP" ] && echo "✅ ($RESOLVED)" || echo "❌ (resolve ke $RESOLVED, IP kita $MY_IP)"

echo -n "HTTP redirect ke HTTPS: "
  REDIRECT=$(curl -sI http://$DOMAIN | grep -i location | tr -d '\r')
  echo "$REDIRECT" | grep -q "https" && echo "✅" || echo "❌ ($REDIRECT)"

echo -n "HTTPS accessible: "
  curl -sf https://$DOMAIN | grep -qi "wordpress\|DOCTYPE" && echo "✅" || echo "❌"

echo -n "SSL certificate valid: "
  EXPIRY=$(echo | openssl s_client -connect $DOMAIN:443 -servername $DOMAIN 2>/dev/null | \
    openssl x509 -noout -enddate 2>/dev/null | cut -d= -f2)
  [ -n "$EXPIRY" ] && echo "✅ (berlaku hingga: $EXPIRY)" || echo "❌"

echo -n "HSTS header ada: "
  curl -sI https://$DOMAIN | grep -qi "strict-transport-security" && echo "✅" || echo "⚠️ (opsional)"

echo -n "WordPress admin accessible: "
  curl -sf https://$DOMAIN/wp-admin/ | grep -qi "login\|dashboard" && echo "✅" || echo "❌"

echo ""
echo "=== SSL Grade Check ==="
echo "Cek SSL grade di: https://www.ssllabs.com/ssltest/analyze.html?d=$DOMAIN"
echo "Target: Grade A atau A+"
```

---

## TAHAP 8: Optimasi Performance

```bash
# Install NGINX PageSpeed module (opsional, perlu compile)
# Alternatif: aktifkan gzip + caching

# Aktifkan gzip di NGINX
sudo tee /etc/nginx/conf.d/gzip.conf << 'EOF'
gzip on;
gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_min_length 256;
gzip_types
    text/plain
    text/css
    text/xml
    text/javascript
    application/json
    application/javascript
    application/xml+rss
    application/atom+xml
    image/svg+xml
    font/woff
    font/woff2;
EOF

sudo nginx -t && sudo systemctl reload nginx

# Install plugin caching WordPress (setelah login ke admin):
# WP Super Cache atau W3 Total Cache atau LiteSpeed Cache
# Pengaturan → WP Super Cache → Enable Caching
```

---

## TAHAP 9: Monitoring Uptime Gratis

```bash
# Daftar di layanan monitoring gratis:
# - UptimeRobot: https://uptimerobot.com (50 monitor gratis)
# - Better Uptime: https://betteruptime.com
# - Freshping: https://freshping.io

# Setup monitoring endpoint
curl -s https://api.uptimerobot.com/v2/newMonitor \
  -d "api_key=YOUR_API_KEY" \
  -d "friendly_name=WordPress Blog" \
  -d "url=https://contoh.com" \
  -d "type=1" \
  -d "interval=5"

# Script cek uptime sederhana
cat > ~/check-uptime.sh << 'EOF'
#!/bin/bash
DOMAIN="${1:-contoh.com}"
URL="https://$DOMAIN"

HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" --max-time 10 "$URL")
RESPONSE_TIME=$(curl -s -o /dev/null -w "%{time_total}" --max-time 10 "$URL")

echo "[$(date '+%Y-%m-%d %H:%M:%S')] $DOMAIN | Status: $HTTP_CODE | Time: ${RESPONSE_TIME}s"

if [ "$HTTP_CODE" != "200" ]; then
    echo "⚠️ ALERT: $DOMAIN mengembalikan HTTP $HTTP_CODE!"
    # Kirim notifikasi (Slack/Telegram/email)
fi
EOF

chmod +x ~/check-uptime.sh

# Jalankan setiap 5 menit via crontab
(crontab -l 2>/dev/null; echo "*/5 * * * * ~/check-uptime.sh contoh.com >> ~/uptime.log 2>&1") | crontab -
```

---

## TAHAP 10: Checklist Production Deployment

```bash
cat << 'EOF'
╔═══════════════════════════════════════════════════════════╗
║         CHECKLIST PRODUCTION WORDPRESS                    ║
╠═══════════════════════════════════════════════════════════╣
║                                                           ║
║  SERVER & JARINGAN                                        ║
║  ☐ VPS dengan IP publik aktif                            ║
║  ☐ Firewall: hanya port 22, 80, 443 terbuka              ║
║  ☐ fail2ban aktif untuk proteksi brute force             ║
║  ☐ SSH key-based authentication (bukan password)         ║
║                                                           ║
║  DNS & DOMAIN                                             ║
║  ☐ Domain terdaftar dan masih aktif                      ║
║  ☐ DNS A record mengarah ke IP VPS                       ║
║  ☐ www subdomain terkonfigurasi                          ║
║  ☐ DNS propagasi selesai (nslookup benar)                ║
║                                                           ║
║  SSL/HTTPS                                                ║
║  ☐ Sertifikat SSL valid (Let's Encrypt)                  ║
║  ☐ HTTP otomatis redirect ke HTTPS                       ║
║  ☐ HSTS header aktif                                     ║
║  ☐ Auto-renewal certbot aktif                            ║
║  ☐ SSL Labs grade A atau A+                              ║
║                                                           ║
║  WORDPRESS                                                ║
║  ☐ URL WordPress dan Home sudah HTTPS                    ║
║  ☐ wp-config.php permission 400                          ║
║  ☐ DISALLOW_FILE_EDIT = true                             ║
║  ☐ Debug mode OFF                                        ║
║  ☐ File editor dinonaktifkan                             ║
║  ☐ Plugin keamanan aktif (Wordfence / iThemes)          ║
║  ☐ User admin bukan username "admin"                     ║
║  ☐ Password admin kuat (minimal 16 karakter)             ║
║                                                           ║
║  BACKUP                                                   ║
║  ☐ Backup database otomatis (harian)                     ║
║  ☐ Backup files otomatis (mingguan)                      ║
║  ☐ Backup tersimpan di lokasi terpisah (S3/B2)           ║
║  ☐ Restore pernah ditest                                 ║
║                                                           ║
║  MONITORING                                               ║
║  ☐ Uptime monitoring aktif                               ║
║  ☐ Alert saat site down                                  ║
║  ☐ Log access dan error terpantau                        ║
║                                                           ║
╚═══════════════════════════════════════════════════════════╝
EOF
```

---

## Troubleshooting

### DNS belum propagasi
```bash
# Cek dari berbagai DNS resolver
dig @8.8.8.8 contoh.com +short
dig @1.1.1.1 contoh.com +short

# Gunakan online tool:
# https://www.whatsmydns.net/
# https://dnschecker.org/
```

### Certbot gagal: "Connection refused"
```bash
# Pastikan port 80 terbuka dan NGINX berjalan
sudo ufw status | grep 80
curl http://localhost
sudo systemctl status nginx
```

### Certbot gagal: "DNS problem"
```bash
# Pastikan DNS sudah propagasi ke IP yang benar
dig contoh.com A +short    # Harus = IP VPS Anda
curl -s ifconfig.me         # IP VPS Anda
```

### WordPress masih HTTP setelah setup HTTPS
```bash
# Tambahkan di wp-config.php (sebelum "That's all"):
define('FORCE_SSL_ADMIN', true);
if (strpos($_SERVER['HTTP_X_FORWARDED_PROTO'], 'https') !== false) {
    $_SERVER['HTTPS'] = 'on';
}
```

### Mixed content warning (gambar/CSS masih HTTP)
```bash
# Install plugin "Really Simple SSL" untuk WordPress
# Atau jalankan search-replace:
wp search-replace 'http://contoh.com' 'https://contoh.com' --all-tables
```

### Sertifikat expired (setelah 90 hari)
```bash
# Renewal manual
sudo certbot renew --force-renewal

# Cek status renewal timer
sudo systemctl status snap.certbot.renew.timer
```

---

## Verifikasi Akhir — Semua Lab Case Study

```bash
DOMAIN="contoh.com"

echo "╔══════════════════════════════════════╗"
echo "║   VERIFIKASI AKHIR: WORDPRESS ONLINE  ║"
echo "╚══════════════════════════════════════╝"
echo ""

echo "=== INFRASTRUKTUR ==="
echo -n "Server berjalan: "
  uptime | grep -q "up" && echo "✅" || echo "❌"
echo -n "Firewall aktif: "
  sudo ufw status | grep -q "active" && echo "✅" || echo "❌"

echo ""
echo "=== DNS ==="
echo -n "Domain resolve: "
  dig +short $DOMAIN | grep -q "[0-9]" && echo "✅ ($(dig +short $DOMAIN))" || echo "❌"

echo ""
echo "=== SSL/HTTPS ==="
echo -n "HTTPS accessible: "
  curl -sf --max-time 10 https://$DOMAIN > /dev/null 2>&1 && echo "✅" || echo "❌"
echo -n "HTTP redirect: "
  curl -sI http://$DOMAIN 2>/dev/null | grep -qi "301\|302" && echo "✅" || echo "❌"
echo -n "Cert valid: "
  echo | openssl s_client -connect $DOMAIN:443 -servername $DOMAIN 2>/dev/null | \
  openssl x509 -noout -checkend 0 2>/dev/null && echo "✅" || echo "❌"

echo ""
echo "=== WORDPRESS ==="
echo -n "Homepage load: "
  curl -sf --max-time 15 https://$DOMAIN | grep -qi "DOCTYPE" && echo "✅" || echo "❌"
echo -n "Admin page: "
  curl -sf --max-time 15 https://$DOMAIN/wp-admin/ | grep -qi "login\|dashboard" && echo "✅" || echo "❌"

echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "🌐 Buka di browser: https://$DOMAIN"
echo "🔑 Admin: https://$DOMAIN/wp-admin"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
```

---

## Checkpoint ✅

- [ ] VPS dengan IP publik siap dan firewall terkonfigurasi
- [ ] Domain dibeli dan DNS record diarahkan ke IP VPS
- [ ] DNS propagasi berhasil (domain resolve ke IP VPS)
- [ ] Certbot berhasil menerbitkan sertifikat Let's Encrypt
- [ ] NGINX dikonfigurasi dengan HTTPS dan redirect otomatis HTTP→HTTPS
- [ ] WordPress URL diperbarui ke `https://`
- [ ] SSL Labs menampilkan grade A atau A+
- [ ] Auto-renewal Certbot aktif
- [ ] Uptime monitoring dikonfigurasi
- [ ] Backup otomatis berjalan
- [ ] Semua item checklist production terpenuhi
- [ ] `https://domain-anda.com` bisa diakses dari HP/laptop siapapun 🎉
