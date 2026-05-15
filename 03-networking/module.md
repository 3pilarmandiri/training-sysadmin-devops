# BAB 3 — Networking untuk DevOps

## Tujuan Pembelajaran

Setelah menyelesaikan bab ini, peserta mampu:
- Memahami model TCP/IP dan cara kerja komunikasi jaringan
- Menghitung subnet dan CIDR notation
- Mengkonfigurasi DNS dan memahami alur resolusi domain
- Mengimplementasikan reverse proxy dengan NGINX

---

## 3.1 Model TCP/IP

TCP/IP adalah fondasi internet modern. Terdiri dari 4 layer:

```
┌─────────────────────────────────────────────┐
│  LAYER 4: APPLICATION                       │
│  HTTP, HTTPS, SSH, FTP, DNS, SMTP           │
├─────────────────────────────────────────────┤
│  LAYER 3: TRANSPORT                         │
│  TCP (reliable) | UDP (fast, unreliable)    │
├─────────────────────────────────────────────┤
│  LAYER 2: INTERNET                          │
│  IP, ICMP, ARP                              │
├─────────────────────────────────────────────┤
│  LAYER 1: NETWORK ACCESS                    │
│  Ethernet, Wi-Fi, MAC Address               │
└─────────────────────────────────────────────┘
```

### TCP vs UDP

| Aspek | TCP | UDP |
|---|---|---|
| Koneksi | Connection-oriented (handshake) | Connectionless |
| Urutan paket | Terjamin | Tidak terjamin |
| Keandalan | Retransmit jika hilang | Tidak ada retransmit |
| Kecepatan | Lebih lambat | Lebih cepat |
| Penggunaan | HTTP, SSH, database | DNS, video streaming, VoIP |

### Three-Way Handshake TCP

```
Client                     Server
  │──── SYN ──────────────→│   (Saya ingin konek)
  │←─── SYN-ACK ──────────│   (Oke, konek)
  │──── ACK ──────────────→│   (Konfirmasi)
  │                         │
  │══════ DATA TRANSFER ════│
  │                         │
  │──── FIN ──────────────→│   (Selesai)
  │←─── FIN-ACK ──────────│   (Oke, tutup)
```

---

## 3.2 IP Address dan CIDR

### IPv4

IPv4 terdiri dari 32 bit, ditulis sebagai 4 oktet desimal:

```
192.168.10.25
│   │   │  │
│   │   │  └── 0-255
│   │   └───── 0-255
│   └───────── 0-255
└───────────── 0-255
```

### Kelas IP Address

| Kelas | Range | Default Mask | Penggunaan |
|---|---|---|---|
| A | 1.0.0.0 – 126.0.0.0 | /8 | Jaringan sangat besar |
| B | 128.0.0.0 – 191.255.0.0 | /16 | Jaringan menengah |
| C | 192.0.0.0 – 223.255.255.0 | /24 | Jaringan kecil |

### IP Privat (RFC 1918)

```
10.0.0.0/8         (10.0.0.0 – 10.255.255.255)
172.16.0.0/12      (172.16.0.0 – 172.31.255.255)
192.168.0.0/16     (192.168.0.0 – 192.168.255.255)
```

### CIDR Notation

CIDR (Classless Inter-Domain Routing) menyatakan subnet dalam format `IP/prefix`:

| CIDR | Subnet Mask | Jumlah Host |
|---|---|---|
| /8 | 255.0.0.0 | 16,777,214 |
| /16 | 255.255.0.0 | 65,534 |
| /24 | 255.255.255.0 | 254 |
| /25 | 255.255.255.128 | 126 |
| /28 | 255.255.255.240 | 14 |
| /30 | 255.255.255.252 | 2 |
| /32 | 255.255.255.255 | 1 (host tunggal) |

**Contoh subnet 192.168.1.0/24:**
```
Network Address  : 192.168.1.0
Broadcast        : 192.168.1.255
Usable Hosts     : 192.168.1.1 – 192.168.1.254
Jumlah Host      : 254
```

---

## 3.3 DNS — Domain Name System

DNS menerjemahkan nama domain menjadi IP address.

### Alur Resolusi DNS

```
Browser: "Apa IP-nya google.com?"
        ↓
   DNS Resolver (ISP / 8.8.8.8)
        ↓
   Root Server (.) → "Tanya .com server"
        ↓
   TLD Server (.com) → "Tanya google.com nameserver"
        ↓
   Authoritative NS (ns1.google.com) → "IP: 142.250.x.x"
        ↓
   DNS Resolver cache hasilnya (TTL)
        ↓
   Browser → 142.250.x.x:443
```

### Tipe Record DNS

| Record | Fungsi | Contoh |
|---|---|---|
| `A` | Domain → IPv4 | `app.example.com → 1.2.3.4` |
| `AAAA` | Domain → IPv6 | `app.example.com → ::1` |
| `CNAME` | Alias ke domain lain | `www → app.example.com` |
| `MX` | Mail server | `@ → mail.example.com` |
| `NS` | Nameserver domain | `@ → ns1.example.com` |
| `TXT` | Teks bebas (SPF, DKIM) | `"v=spf1 include:..."` |
| `PTR` | IP → Domain (reverse) | `1.2.3.4 → app.example.com` |
| `SRV` | Service discovery | `_http._tcp → ...` |

### Perintah DNS di Linux

```bash
# Query DNS
nslookup google.com
dig google.com
dig google.com A          # Hanya record A
dig google.com MX         # Record MX
dig +short google.com     # Hanya IP-nya

# Reverse lookup (IP ke domain)
dig -x 8.8.8.8

# Query ke DNS server spesifik
dig @8.8.8.8 google.com
dig @1.1.1.1 google.com

# Cek TTL
dig +nocmd google.com A +noall +answer

# File hosts lokal (override DNS)
cat /etc/hosts
echo "127.0.0.1 myapp.local" | sudo tee -a /etc/hosts
```

---

## 3.4 Port dan Firewall

### Port Penting yang Harus Dihafal

| Port | Protokol | Service |
|---|---|---|
| 22 | TCP | SSH |
| 53 | TCP/UDP | DNS |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 3306 | TCP | MySQL |
| 5432 | TCP | PostgreSQL |
| 6379 | TCP | Redis |
| 8080 | TCP | HTTP alt / App server |
| 9090 | TCP | Prometheus |
| 3000 | TCP | Grafana |

### UFW (Uncomplicated Firewall)

```bash
# Status firewall
sudo ufw status verbose

# Enable firewall
sudo ufw enable

# Allow / deny port
sudo ufw allow 22/tcp          # Allow SSH
sudo ufw allow 80/tcp          # Allow HTTP
sudo ufw allow 443/tcp         # Allow HTTPS
sudo ufw deny 3306/tcp         # Blokir MySQL dari luar

# Allow dari IP spesifik
sudo ufw allow from 192.168.1.0/24 to any port 22

# Hapus rule
sudo ufw delete allow 80/tcp

# Reset semua rule
sudo ufw reset
```

### ss dan netstat — Cek Port Aktif

```bash
# Tampilkan port yang sedang listen
ss -tlnp
ss -ulnp    # UDP

# Cek port spesifik
ss -tlnp | grep :80
ss -tlnp | grep :443

# Cek koneksi aktif
ss -tnp

# Verifikasi port terbuka dari luar
nc -zv localhost 80
nc -zv 192.168.1.1 22
```

---

## 3.5 Reverse Proxy dengan NGINX

### Apa itu Reverse Proxy?

```
CLIENT REQUEST
       ↓
  ┌─────────────────────────────────────┐
  │          NGINX (Port 80/443)        │  ← Reverse Proxy
  │                                     │
  │  /api/*   → Backend App :8080       │
  │  /static/ → Static Files :8081     │
  │  /*       → Frontend App :3000     │
  └─────────────────────────────────────┘
       ↓
  Backend Services (tidak terekspos langsung)
```

**Keuntungan Reverse Proxy:**
- Satu IP/port publik untuk banyak service
- SSL termination (HTTPS di satu tempat)
- Load balancing
- Caching
- Proteksi backend

### Konfigurasi NGINX Dasar

```nginx
# /etc/nginx/nginx.conf

user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
    worker_connections 1024;
    multi_accept on;
}

http {
    # MIME types
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging
    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent"';
    access_log /var/log/nginx/access.log main;
    error_log  /var/log/nginx/error.log warn;

    # Performance
    sendfile on;
    tcp_nopush on;
    keepalive_timeout 65;
    gzip on;

    # Virtual hosts
    include /etc/nginx/sites-enabled/*;
}
```

### Reverse Proxy ke Aplikasi Backend

```nginx
# /etc/nginx/sites-available/myapp.conf

upstream backend {
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;  # Load balance ke 2 instance
    keepalive 32;
}

server {
    listen 80;
    server_name app.example.com;

    # Redirect HTTP ke HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name app.example.com;

    # SSL Certificate
    ssl_certificate     /etc/ssl/certs/app.crt;
    ssl_certificate_key /etc/ssl/private/app.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    add_header X-XSS-Protection "1; mode=block";

    # Proxy ke backend
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 60s;
        proxy_read_timeout 60s;
    }

    # Static files langsung dari NGINX
    location /static/ {
        root /var/www/myapp;
        expires 30d;
        add_header Cache-Control "public";
    }

    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

### Perintah Manajemen NGINX

```bash
# Test konfigurasi (sebelum apply)
sudo nginx -t

# Reload config tanpa downtime
sudo systemctl reload nginx

# Restart NGINX
sudo systemctl restart nginx

# Lihat log real-time
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log

# Enable site
sudo ln -s /etc/nginx/sites-available/myapp.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

---

## 3.6 Load Balancing dengan NGINX

```nginx
upstream api_servers {
    # Round Robin (default)
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080;
}

upstream api_weighted {
    # Weighted (server kuat dapat beban lebih)
    server 10.0.0.1:8080 weight=3;
    server 10.0.0.2:8080 weight=1;
}

upstream api_leastconn {
    # Least connections (kirim ke server paling idle)
    least_conn;
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}

upstream api_iphash {
    # IP Hash (satu client selalu ke server sama)
    ip_hash;
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}
```

---

## Ringkasan

| Konsep | Inti |
|---|---|
| TCP/IP | 4-layer model: Application, Transport, Internet, Network Access |
| CIDR | Notasi subnet: `192.168.1.0/24` = 254 host |
| DNS | Menerjemahkan domain ke IP; record A, CNAME, MX, TXT |
| Reverse Proxy | NGINX sebagai gateway tunggal untuk banyak backend |
| Load Balancing | Distribusi traffic: round-robin, weighted, least-conn |
