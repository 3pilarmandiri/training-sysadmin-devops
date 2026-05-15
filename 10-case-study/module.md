# BAB 10 — Studi Kasus: Deploy WordPress Sampai Online

## Tujuan Pembelajaran

Studi kasus ini mengintegrasikan seluruh kompetensi yang telah dipelajari di modul sebelumnya dalam satu proyek nyata: **men-deploy WordPress dari nol hingga bisa diakses dari internet**.

Setelah menyelesaikan studi kasus ini, peserta mampu:
- Menginstal WordPress di stack LEMP (Linux + NGINX + MySQL + PHP) secara tradisional
- Mengkontainerisasi WordPress menggunakan Docker Compose
- Mengkonfigurasi NGINX sebagai reverse proxy di depan WordPress
- Mengamankan site dengan HTTPS menggunakan Let's Encrypt / Certbot
- Menghubungkan domain ke server
- Melakukan hardening keamanan dasar WordPress

---

## Mengapa WordPress?

WordPress digunakan oleh **43% website di seluruh dunia** (per 2024). Kemampuan men-deploy dan mengelola WordPress di infrastruktur sendiri adalah skill wajib bagi seorang DevOps/Sysadmin.

Studi kasus ini mencerminkan pekerjaan nyata yang sering ditemui:
- Client meminta website/blog → deploy WordPress
- Startup butuh CMS → setup WordPress + custom domain
- E-commerce sederhana → WooCommerce di atas WordPress

---

## Arsitektur yang Dibangun

### Opsi A — LEMP Stack (Tradisional)

```
Internet
    │
    ▼  Port 80/443
 NGINX (Reverse Proxy + SSL Termination)
    │
    ▼  PHP-FPM socket
 PHP 8.2-FPM (WordPress)
    │
    ▼  Port 3306 (lokal saja)
 MySQL 8.0 (Database)
    │
 /var/www/html/wordpress (File System)
```

**Kelebihan:** Performa tinggi, kontrol penuh, cocok untuk server dedicated.

### Opsi B — Docker Compose (Modern)

```
Internet
    │
    ▼  Port 80/443
 NGINX (Container — Reverse Proxy + SSL)
    │
    ▼  Port 9000 (PHP-FPM)
 WordPress (Container — PHP-FPM)
    │
    ▼  Port 3306
 MySQL (Container — Database)
    │
 Volume: wp_data, db_data
```

**Kelebihan:** Portabel, mudah backup, mudah di-migrate, isolation baik.

---

## Keterkaitan dengan Modul Sebelumnya

| Lab | Kompetensi yang Digunakan |
|---|---|
| Lab 13 — LEMP Stack | Modul 02 (Linux), Modul 03 (NGINX), Modul 02 (permission/user) |
| Lab 14 — Docker Compose | Modul 05 (Docker), Modul 03 (NGINX), Modul 04 (Git) |
| Lab 15 — Go Online | Modul 03 (NGINX/DNS), Modul 06 (otomasi), Modul 09 (security) |

---

## Persiapan

### Kebutuhan Server

| Kebutuhan | Minimum | Rekomendasi |
|---|---|---|
| CPU | 1 core | 2 core |
| RAM | 1 GB | 2 GB |
| Disk | 20 GB | 40 GB |
| OS | Ubuntu 22.04 | Ubuntu 22.04 LTS |
| Koneksi | Ada internet | IP publik (untuk Lab 15) |

### Untuk Lab 15 (Go Online), tambahan:

- Domain yang sudah terdaftar (bisa beli di Niagahoster, Namecheap, dll)
- IP publik yang bisa di-akses dari internet (VPS: DigitalOcean, IDCloudHost, Biznet Gio, dll)
- Port 80 dan 443 terbuka di firewall

---

## Pilih Jalur Belajar

```
Jalur Lengkap (Semua Lab):
  Lab 13 → Lab 14 → Lab 15
  (3–4 jam, paling lengkap)

Jalur Cepat Modern (Docker):
  Lab 14 → Lab 15
  (2–3 jam)

Jalur Tradisional (Sysadmin):
  Lab 13 → Lab 15 (skip Docker)
  (2–3 jam)
```

---

## Referensi

- [WordPress Requirements](https://wordpress.org/about/requirements/)
- [NGINX WordPress Config](https://www.nginx.com/resources/wiki/start/topics/recipes/wordpress/)
- [Certbot Documentation](https://certbot.eff.org/)
- [WordPress Security Hardening](https://wordpress.org/documentation/article/hardening-wordpress/)
