# LAB 13 — Install WordPress dengan LEMP Stack

## Tujuan
Menginstal WordPress secara tradisional di atas stack LEMP (Linux + NGINX + MySQL + PHP) pada server Ubuntu, mulai dari instalasi paket hingga WordPress siap digunakan.

## Prasyarat
- Ubuntu 22.04 LTS dengan akses sudo
- Lab 01 (environment), Lab 02 (user management), Lab 04 (NGINX) selesai
- Koneksi internet

## Estimasi Waktu
60 – 90 menit

---

## Arsitektur

```
Browser
  │
  ▼ HTTP :80 / HTTPS :443
NGINX (Web Server + Reverse Proxy)
  │
  ▼ via Unix socket: /run/php/php8.2-fpm.sock
PHP 8.2-FPM (menjalankan kode PHP WordPress)
  │
  ▼ TCP :3306 (lokal saja)
MySQL 8.0 (database WordPress)
  │
/var/www/wordpress/ (file WordPress)
```

---

## TAHAP 1: Install MySQL

### Langkah 1.1: Install MySQL Server

```bash
sudo apt update
sudo apt install -y mysql-server

# Start dan enable MySQL
sudo systemctl enable --now mysql

# Cek status
sudo systemctl status mysql
```

### Langkah 1.2: Amankan Instalasi MySQL

```bash
sudo mysql_secure_installation
```

Jawab prompt berikut:
```
VALIDATE PASSWORD component? → N (untuk lab, di prod → Y)
New password: → WordPressDB@2024
Re-enter new password: → WordPressDB@2024
Remove anonymous users? → Y
Disallow root login remotely? → Y
Remove test database? → Y
Reload privilege tables now? → Y
```

### Langkah 1.3: Buat Database dan User WordPress

```bash
sudo mysql -u root

# Di dalam MySQL shell:
```

```sql
-- Buat database
CREATE DATABASE wordpress_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Buat user khusus WordPress (jangan pakai root!)
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'WPUserPass@2024';

-- Beri hak akses
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wp_user'@'localhost';

-- Refresh privileges
FLUSH PRIVILEGES;

-- Verifikasi
SHOW DATABASES;
SELECT User, Host FROM mysql.user;

EXIT;
```

Verifikasi dari terminal:
```bash
mysql -u wp_user -p wordpress_db
# Masukkan: WPUserPass@2024
# Harus bisa masuk → EXIT;
```

---

## TAHAP 2: Install PHP 8.2

### Langkah 2.1: Tambah Repository PHP

```bash
sudo apt install -y software-properties-common
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update
```

### Langkah 2.2: Install PHP dan Ekstensi WordPress

```bash
sudo apt install -y \
  php8.2-fpm \
  php8.2-mysql \
  php8.2-curl \
  php8.2-gd \
  php8.2-mbstring \
  php8.2-xml \
  php8.2-xmlrpc \
  php8.2-zip \
  php8.2-intl \
  php8.2-bcmath \
  php8.2-imagick \
  php8.2-soap \
  php8.2-opcache

# Start PHP-FPM
sudo systemctl enable --now php8.2-fpm
sudo systemctl status php8.2-fpm
```

### Langkah 2.3: Konfigurasi PHP untuk WordPress

```bash
sudo vim /etc/php/8.2/fpm/php.ini
```

Cari dan ubah nilai berikut (gunakan `/` untuk search di vim):
```ini
upload_max_filesize = 64M
post_max_size = 64M
memory_limit = 256M
max_execution_time = 300
max_input_time = 300
max_input_vars = 3000
date.timezone = Asia/Jakarta
```

```bash
# Restart PHP-FPM setelah perubahan config
sudo systemctl restart php8.2-fpm
```

---

## TAHAP 3: Install NGINX

### Langkah 3.1: Install NGINX

```bash
sudo apt install -y nginx

sudo systemctl enable --now nginx
sudo systemctl status nginx

# Test NGINX berjalan
curl -I http://localhost
# HTTP/1.1 200 OK
```

### Langkah 3.2: Konfigurasi NGINX untuk WordPress

```bash
# Hapus config default
sudo rm /etc/nginx/sites-enabled/default

# Buat config untuk WordPress
sudo tee /etc/nginx/sites-available/wordpress.conf << 'EOF'
server {
    listen 80;
    listen [::]:80;

    # Ganti dengan domain atau IP Anda
    server_name wordpress.local;

    root /var/www/wordpress;
    index index.php index.html index.htm;

    # Log
    access_log /var/log/nginx/wordpress.access.log;
    error_log  /var/log/nginx/wordpress.error.log;

    # Ukuran upload maksimal
    client_max_body_size 64M;

    # WordPress permalink dan rewrite
    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    # Proses file PHP via PHP-FPM
    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_read_timeout 300;
        fastcgi_buffers 16 16k;
        fastcgi_buffer_size 32k;
    }

    # Cache static files
    location ~* \.(css|js|jpg|jpeg|png|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 30d;
        add_header Cache-Control "public, no-transform";
        access_log off;
    }

    # Blokir akses ke file sensitif
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }

    location ~* /(?:uploads|files)/.*\.php$ {
        deny all;
    }

    location = /xmlrpc.php {
        deny all;
        access_log off;
        log_not_found off;
    }

    # WordPress health check
    location = /wp-login.php {
        # Batasi akses login (uncomment untuk production)
        # allow 1.2.3.4;  # IP Anda
        # deny all;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
EOF

# Enable site
sudo ln -s /etc/nginx/sites-available/wordpress.conf \
           /etc/nginx/sites-enabled/wordpress.conf

# Test config
sudo nginx -t

# Reload NGINX
sudo systemctl reload nginx
```

---

## TAHAP 4: Download dan Setup WordPress

### Langkah 4.1: Download WordPress

```bash
cd /tmp

# Download WordPress versi terbaru
wget https://wordpress.org/latest.tar.gz

# Atau jika wget lambat, coba curl
# curl -LO https://wordpress.org/latest.tar.gz

# Ekstrak
tar -xzf latest.tar.gz

ls wordpress/    # Pastikan ada index.php, wp-config-sample.php, dll
```

### Langkah 4.2: Pindah ke Web Root

```bash
# Pindahkan ke direktori web
sudo mv wordpress /var/www/wordpress

# Set ownership ke www-data (user NGINX/PHP-FPM)
sudo chown -R www-data:www-data /var/www/wordpress

# Set permission yang aman
sudo find /var/www/wordpress -type d -exec chmod 755 {} \;
sudo find /var/www/wordpress -type f -exec chmod 644 {} \;

# Verifikasi
ls -la /var/www/wordpress
```

### Langkah 4.3: Buat wp-config.php

```bash
# Copy sample config
sudo cp /var/www/wordpress/wp-config-sample.php \
        /var/www/wordpress/wp-config.php

# Edit konfigurasi database
sudo sed -i "s/database_name_here/wordpress_db/" \
  /var/www/wordpress/wp-config.php
sudo sed -i "s/username_here/wp_user/" \
  /var/www/wordpress/wp-config.php
sudo sed -i "s/password_here/WPUserPass@2024/" \
  /var/www/wordpress/wp-config.php

# Ganti 'localhost' jika MySQL ada di host lain
# sudo sed -i "s/localhost/127.0.0.1/" /var/www/wordpress/wp-config.php
```

Generate secret keys (sangat penting untuk keamanan):

```bash
# Ambil secret keys dari WordPress API
KEYS=$(curl -s https://api.wordpress.org/secret-key/1.1/salt/)

# Simpan ke file sementara
echo "$KEYS" > /tmp/wp-keys.txt
cat /tmp/wp-keys.txt
```

Edit `wp-config.php` dan ganti blok `AUTH_KEY` hingga `NONCE_SALT` dengan output di atas:

```bash
sudo vim /var/www/wordpress/wp-config.php
# Cari baris: define('AUTH_KEY', ...
# Hapus 8 baris tersebut dan ganti dengan output dari curl di atas
```

Tambahkan konfigurasi ekstra di akhir blok keys:

```bash
sudo tee -a /var/www/wordpress/wp-config.php > /dev/null << 'EOF'

/** Konfigurasi Tambahan */
define('WP_DEBUG', false);
define('WP_DEBUG_LOG', false);
define('WP_DEBUG_DISPLAY', false);

// Batasi jumlah revisi post
define('WP_POST_REVISIONS', 5);

// Hapus trash otomatis setelah 30 hari
define('EMPTY_TRASH_DAYS', 30);

// Nonaktifkan file editor di admin (keamanan)
define('DISALLOW_FILE_EDIT', true);
define('DISALLOW_FILE_MODS', false);  // Set true jika tidak mau install plugin

// Direktori upload
define('UPLOADS', 'wp-content/uploads');

// Timezone
date_default_timezone_set('Asia/Jakarta');
EOF
```

---

## TAHAP 5: Konfigurasi Sistem

### Langkah 5.1: Setup /etc/hosts untuk Akses Lokal

```bash
# Tambahkan domain lokal
echo "127.0.0.1 wordpress.local" | sudo tee -a /etc/hosts

# Verifikasi
ping -c 1 wordpress.local
```

### Langkah 5.2: Konfigurasi Firewall

```bash
# Allow HTTP dan HTTPS
sudo ufw allow 'Nginx Full'
sudo ufw allow OpenSSH
sudo ufw enable

# Verifikasi
sudo ufw status
```

### Langkah 5.3: Buat Direktori Upload WordPress

```bash
sudo mkdir -p /var/www/wordpress/wp-content/uploads
sudo chown www-data:www-data /var/www/wordpress/wp-content/uploads
sudo chmod 755 /var/www/wordpress/wp-content/uploads
```

---

## TAHAP 6: Wizard Instalasi WordPress

### Langkah 6.1: Akses Wizard via Browser

Buka browser dan kunjungi:
```
http://wordpress.local
```

Anda akan disambut halaman wizard instalasi WordPress.

**Jika berjalan di server remote (VPS), akses via:**
```
http://IP_SERVER_ANDA
```

### Langkah 6.2: Ikuti Wizard Instalasi

**Halaman 1 — Pilih Bahasa:**
- Pilih `Bahasa Indonesia` atau `English (United States)`
- Klik **Lanjutkan**

**Halaman 2 — Informasi Situs:**

| Field | Isi |
|---|---|
| Judul Situs | `DevOps Training Blog` |
| Nama pengguna | `admin` |
| Kata sandi | Buat password kuat (minimal 12 karakter) |
| Email Anda | `admin@devops.local` |
| Visibilitas mesin pencari | Biarkan tercentang untuk development |

- Klik **Install WordPress**

**Halaman 3 — Sukses!**
- Klik **Masuk** → login dengan username dan password yang baru dibuat

### Langkah 6.3: Verifikasi Dashboard

Setelah login, Anda akan melihat WordPress Dashboard. Cek:
- Kunjungi situs: `http://wordpress.local`
- Dashboard admin: `http://wordpress.local/wp-admin`

---

## TAHAP 7: Verifikasi Stack

```bash
echo "=== Verifikasi LEMP Stack ==="

echo -n "MySQL berjalan: "
  systemctl is-active mysql | grep -q active && echo "✅" || echo "❌"

echo -n "PHP-FPM berjalan: "
  systemctl is-active php8.2-fpm | grep -q active && echo "✅" || echo "❌"

echo -n "NGINX berjalan: "
  systemctl is-active nginx | grep -q active && echo "✅" || echo "❌"

echo -n "Database wordpress_db ada: "
  mysql -u wp_user -pWPUserPass@2024 wordpress_db -e "SHOW TABLES;" 2>/dev/null | \
  grep -q wp_posts && echo "✅" || echo "❌"

echo -n "WordPress files ada: "
  [ -f /var/www/wordpress/wp-config.php ] && echo "✅" || echo "❌"

echo -n "NGINX config valid: "
  sudo nginx -t 2>&1 | grep -q "ok" && echo "✅" || echo "❌"

echo -n "WordPress homepage accessible: "
  curl -sf http://wordpress.local | grep -q -i "wordpress\|DOCTYPE" && echo "✅" || echo "❌"

echo ""
echo "=== Status Service ==="
systemctl status mysql php8.2-fpm nginx --no-pager | grep -E "Active:|●"
```

---

## TAHAP 8: Hardening Keamanan Dasar

```bash
# 1. Proteksi wp-config.php
sudo chmod 400 /var/www/wordpress/wp-config.php
sudo chown root:www-data /var/www/wordpress/wp-config.php

# 2. Buat file .htaccess placeholder
sudo touch /var/www/wordpress/.htaccess
sudo chown www-data:www-data /var/www/wordpress/.htaccess
sudo chmod 644 /var/www/wordpress/.htaccess

# 3. Ubah tabel prefix default (jika belum diubah saat install)
# Tambahkan ke wp-config.php: $table_prefix = 'wp_training_';
# (harus dilakukan SEBELUM install)

# 4. Cek file permission
echo "=== Cek Permission Sensitif ==="
stat -c "%a %n" /var/www/wordpress/wp-config.php
stat -c "%a %n" /var/www/wordpress/wp-content
stat -c "%a %n" /var/www/wordpress/wp-content/uploads

# 5. Scan vulnerabilities dasar
echo ""
echo "=== Cek Versi PHP ==="
php8.2 --version

echo ""
echo "=== Cek Versi MySQL ==="
mysql --version

echo ""
echo "=== Cek Versi WordPress ==="
grep "wp_version = " /var/www/wordpress/wp-includes/version.php
```

---

## TAHAP 9: Backup Script Dasar

```bash
cat > ~/backup-wordpress.sh << 'EOF'
#!/bin/bash
# Script backup WordPress sederhana

BACKUP_DIR="/home/$USER/wp-backups"
DATE=$(date +%Y%m%d_%H%M%S)
WP_DIR="/var/www/wordpress"
DB_NAME="wordpress_db"
DB_USER="wp_user"
DB_PASS="WPUserPass@2024"

mkdir -p "$BACKUP_DIR"

echo "=== Backup WordPress - $DATE ==="

# 1. Backup database
echo "Backup database..."
mysqldump -u "$DB_USER" -p"$DB_PASS" "$DB_NAME" | \
  gzip > "$BACKUP_DIR/db_${DATE}.sql.gz"
echo "  ✅ Database: db_${DATE}.sql.gz"

# 2. Backup files (kecuali cache)
echo "Backup files..."
tar -czf "$BACKUP_DIR/files_${DATE}.tar.gz" \
  --exclude="$WP_DIR/wp-content/cache" \
  --exclude="$WP_DIR/wp-content/uploads/cache" \
  "$WP_DIR"
echo "  ✅ Files: files_${DATE}.tar.gz"

# 3. Hapus backup lebih dari 7 hari
find "$BACKUP_DIR" -mtime +7 -delete
echo "  ✅ Backup lama dibersihkan"

echo ""
echo "Backup selesai! Tersimpan di: $BACKUP_DIR"
ls -lh "$BACKUP_DIR"
EOF

chmod +x ~/backup-wordpress.sh
echo "Script backup siap: ~/backup-wordpress.sh"
```

---

## Troubleshooting

### Error: "Error establishing a database connection"
```bash
# Verifikasi kredensial database
mysql -u wp_user -pWPUserPass@2024 wordpress_db -e "SELECT 1;"
grep "DB_" /var/www/wordpress/wp-config.php
```

### Error: "502 Bad Gateway"
```bash
# Cek PHP-FPM berjalan dan socket ada
systemctl status php8.2-fpm
ls -la /run/php/php8.2-fpm.sock
sudo journalctl -u php8.2-fpm -f
```

### Error: "403 Forbidden"
```bash
# Cek permission
ls -la /var/www/wordpress/
sudo chown -R www-data:www-data /var/www/wordpress
```

### WordPress tidak bisa upload gambar
```bash
sudo chown -R www-data:www-data /var/www/wordpress/wp-content/uploads
sudo chmod 755 /var/www/wordpress/wp-content/uploads
```

### PHP max upload size
```bash
# Edit /etc/php/8.2/fpm/php.ini
sudo sed -i 's/upload_max_filesize = .*/upload_max_filesize = 64M/' \
  /etc/php/8.2/fpm/php.ini
sudo systemctl restart php8.2-fpm
```

---

## Checkpoint ✅

- [ ] MySQL terinstall, database `wordpress_db` dan user `wp_user` terbuat
- [ ] PHP 8.2-FPM terinstall dengan semua ekstensi yang dibutuhkan
- [ ] NGINX terinstall dan dikonfigurasi untuk WordPress
- [ ] WordPress berhasil didownload dan diekstrak ke `/var/www/wordpress`
- [ ] `wp-config.php` berisi kredensial database yang benar dan secret keys
- [ ] Wizard instalasi WordPress berhasil diselesaikan
- [ ] Bisa login ke WordPress admin di `http://wordpress.local/wp-admin`
- [ ] Script backup tersedia
