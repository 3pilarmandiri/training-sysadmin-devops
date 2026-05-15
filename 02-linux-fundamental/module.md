# BAB 2 — Linux Fundamental

## Tujuan Pembelajaran

Setelah menyelesaikan bab ini, peserta mampu:
- Menavigasi filesystem Linux dan memahami hierarkinya
- Mengelola user, group, dan permission dengan tepat
- Menggunakan systemctl untuk manajemen service
- Memahami proses, variabel lingkungan, dan shell scripting dasar

---

## 2.1 Hierarki Filesystem Linux (FHS)

Linux mengikuti **Filesystem Hierarchy Standard (FHS)**. Semua dimulai dari root `/`.

```
/
├── bin/        → Binary perintah dasar (ls, cp, mv)
├── boot/       → File bootloader (kernel, initrd)
├── dev/        → Device files (disk, terminal, dll)
├── etc/        → File konfigurasi sistem
├── home/       → Home directory setiap user (/home/alice)
├── lib/        → Library bersama
├── media/      → Mount point media removable
├── mnt/        → Mount point sementara
├── opt/        → Aplikasi third-party opsional
├── proc/       → Virtual filesystem info proses & kernel
├── root/       → Home directory root user
├── run/        → Data runtime (PID files, socket)
├── srv/        → Data service (web, ftp)
├── sys/        → Virtual filesystem info hardware
├── tmp/        → File sementara (terhapus saat reboot)
├── usr/        → Program & data pengguna
│   ├── bin/    → Binary pengguna
│   ├── lib/    → Library pengguna
│   └── local/  → Software yang dikompilasi lokal
└── var/        → Data variabel (log, spool, cache)
    ├── log/    → File log sistem
    └── www/    → Root web server (default)
```

### Navigasi Dasar

```bash
pwd               # Tampilkan direktori saat ini
ls -la            # List isi direktori dengan detail
cd /etc           # Pindah ke direktori /etc
cd ~              # Kembali ke home directory
cd ..             # Naik satu level
cd -              # Kembali ke direktori sebelumnya

# Path absolut vs relatif
ls /etc/nginx          # Absolut (dari root)
ls ../etc/nginx        # Relatif (dari posisi saat ini)
```

---

## 2.2 Manajemen File dan Direktori

```bash
# Buat direktori
mkdir project
mkdir -p project/src/config    # Buat sekaligus parent-nya

# Salin file/direktori
cp file.txt backup.txt
cp -r direktori/ backup/       # Rekursif (untuk direktori)

# Pindah / rename
mv file.txt /tmp/
mv lama.txt baru.txt

# Hapus
rm file.txt
rm -rf direktori/              # Hapus direktori & isinya (hati-hati!)

# Buat file kosong / update timestamp
touch app.log

# Lihat isi file
cat /etc/os-release
less /var/log/syslog           # Navigasi dengan j/k/q
head -20 /var/log/syslog       # 20 baris pertama
tail -f /var/log/syslog        # Ikuti log secara real-time
```

---

## 2.3 User dan Group Management

### Konsep Dasar

Di Linux, setiap **file** dan **proses** dimiliki oleh seorang **user** dan **group**. Kontrol akses diatur berdasarkan kepemilikan ini.

```
/etc/passwd   → Database user (nama, UID, GID, home, shell)
/etc/shadow   → Password user yang di-hash
/etc/group    → Database group (nama, GID, anggota)
```

### Perintah User Management

```bash
# Tambah user baru (dengan home directory dan shell default)
sudo useradd -m -s /bin/bash alice

# Set password user
sudo passwd alice

# Tambah user interaktif (lebih lengkap)
sudo adduser bob

# Modifikasi user
sudo usermod -aG sudo alice      # Tambah alice ke grup sudo
sudo usermod -s /bin/zsh alice   # Ganti shell default
sudo usermod -l newname alice    # Rename user

# Kunci / buka akun user
sudo usermod -L alice            # Lock (nonaktifkan login)
sudo usermod -U alice            # Unlock

# Hapus user
sudo userdel alice               # Hapus user (home tetap ada)
sudo userdel -r alice            # Hapus user + home directory

# Info user saat ini
whoami
id
id alice                         # Info user alice
```

### Perintah Group Management

```bash
# Buat group baru
sudo groupadd devteam

# Tambahkan user ke group
sudo usermod -aG devteam alice
sudo usermod -aG devteam bob

# Hapus user dari group
sudo gpasswd -d alice devteam

# Hapus group
sudo groupdel devteam

# Lihat anggota group
getent group devteam
cat /etc/group | grep devteam

# Tampilkan semua group milik user
groups alice
```

---

## 2.4 Permission dan Ownership

### Struktur Permission

Setiap file/direktori memiliki permission dalam format:

```
-rwxr-xr--  1  alice  devteam  4096  Jan 1 10:00  script.sh
│└┬┘└┬┘└┬┘       │       │
│ │  │  │        │       └── Group pemilik
│ │  │  │        └────────── User pemilik
│ │  │  └─────────────────── Permission others (r--)
│ │  └────────────────────── Permission group (r-x)
│ └───────────────────────── Permission owner (rwx)
└─────────────────────────── Tipe (- = file, d = dir, l = symlink)
```

### Nilai Permission (Oktal)

| Simbolik | Oktal | Arti |
|---|---|---|
| `r` | 4 | Read — baca isi file / list isi direktori |
| `w` | 2 | Write — tulis ke file / buat/hapus di direktori |
| `x` | 1 | Execute — jalankan file / masuk ke direktori |
| `-` | 0 | Tidak ada permission |

**Contoh:**
```
rwxr-xr-- = 7 5 4 = 754
rw-rw-r-- = 6 6 4 = 664
rwx------ = 7 0 0 = 700
```

### Perintah chmod dan chown

```bash
# chmod — ubah permission

# Metode simbolik
chmod u+x script.sh      # Tambah execute untuk owner
chmod g-w file.txt       # Hapus write untuk group
chmod o=r file.txt       # Set others hanya read
chmod a+r file.txt       # Tambah read untuk semua (all)

# Metode numerik (oktal)
chmod 755 script.sh      # rwxr-xr-x (executable oleh semua)
chmod 644 index.html     # rw-r--r-- (web file biasa)
chmod 600 ~/.ssh/id_rsa  # rw------- (private key)
chmod 700 ~/.ssh         # rwx------ (direktori SSH)

# Rekursif
chmod -R 755 /var/www/html

# chown — ubah kepemilikan
sudo chown alice file.txt          # Ganti owner
sudo chown alice:devteam file.txt  # Ganti owner & group
sudo chown -R www-data:www-data /var/www/  # Rekursif

# chgrp — ubah group saja
sudo chgrp devteam file.txt
```

### Special Permission

```bash
# SUID (Set User ID) — jalankan sebagai owner file
chmod u+s /usr/bin/program    # atau chmod 4755

# SGID (Set Group ID) — jalankan sebagai group file
chmod g+s /shared/directory   # atau chmod 2755

# Sticky Bit — hanya owner yang bisa hapus file di direktori
chmod +t /tmp                 # atau chmod 1777
ls -la / | grep tmp           # tampil drwxrwxrwt
```

---

## 2.5 Manajemen Proses

```bash
# Lihat proses berjalan
ps aux                        # Semua proses
ps aux | grep nginx           # Filter proses nginx
pgrep nginx                   # PID proses nginx

# Monitor real-time
top                           # Monitor dasar
htop                          # Monitor interaktif (lebih bagus)

# Kill proses
kill 1234                     # Kirim SIGTERM (graceful)
kill -9 1234                  # Kirim SIGKILL (paksa)
pkill nginx                   # Kill berdasarkan nama
killall nginx

# Background / foreground
sleep 60 &                    # Jalankan di background
jobs                          # List background jobs
fg %1                         # Bawa ke foreground
bg %1                         # Kirim ke background lagi
```

---

## 2.6 Systemctl — Manajemen Service

**systemd** adalah sistem init modern di Linux. `systemctl` adalah antarmuka untuk mengelola service (unit).

```bash
# Status service
sudo systemctl status nginx
sudo systemctl status docker

# Start / Stop / Restart
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl reload nginx      # Reload config tanpa restart

# Enable / Disable (saat boot)
sudo systemctl enable nginx      # Aktif saat boot
sudo systemctl disable nginx     # Tidak aktif saat boot

sudo systemctl enable --now nginx   # Enable + langsung start

# List semua service
systemctl list-units --type=service
systemctl list-units --type=service --state=running

# Lihat log service
journalctl -u nginx              # Log service nginx
journalctl -u nginx -f           # Follow log real-time
journalctl -u nginx --since "1 hour ago"
journalctl -p err                # Hanya error level
```

### Membuat Custom Service

```bash
sudo vim /etc/systemd/system/myapp.service
```

```ini
[Unit]
Description=My Custom App
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/app --config /etc/myapp/config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now myapp
```

---

## 2.7 Variabel Lingkungan (Environment Variables)

```bash
# Tampilkan semua variabel
env
printenv

# Tampilkan satu variabel
echo $PATH
echo $HOME
echo $USER

# Set variabel (hanya untuk sesi saat ini)
export MY_VAR="hello devops"
echo $MY_VAR

# Set permanen (tambahkan ke ~/.bashrc atau /etc/environment)
echo 'export MY_VAR="hello devops"' >> ~/.bashrc
source ~/.bashrc

# Variabel penting yang harus diketahui
echo $PATH      # Daftar direktori pencarian executable
echo $HOME      # Home directory user saat ini
echo $USER      # Nama user saat ini
echo $SHELL     # Shell yang digunakan
echo $HOSTNAME  # Nama host mesin
echo $LANG      # Locale bahasa sistem
```

---

## 2.8 Shell Scripting Dasar

```bash
#!/bin/bash
# Baris pertama disebut "shebang" — menentukan interpreter

# Variabel
NAME="DevOps"
echo "Hello, $NAME!"

# Input dari user
read -p "Masukkan nama Anda: " USERNAME
echo "Selamat datang, $USERNAME!"

# Kondisi
if [ -f "/etc/nginx/nginx.conf" ]; then
    echo "NGINX terinstall"
else
    echo "NGINX belum terinstall"
fi

# Loop
for service in nginx docker ssh; do
    status=$(systemctl is-active $service)
    echo "$service: $status"
done

# Fungsi
check_port() {
    local port=$1
    if ss -tlnp | grep -q ":$port "; then
        echo "Port $port: OPEN"
    else
        echo "Port $port: CLOSED"
    fi
}

check_port 80
check_port 443
check_port 22
```

---

## Ringkasan Perintah Penting

| Kategori | Perintah |
|---|---|
| Navigasi | `pwd`, `ls`, `cd`, `find`, `tree` |
| File | `cat`, `less`, `head`, `tail`, `cp`, `mv`, `rm`, `touch` |
| User | `useradd`, `usermod`, `userdel`, `passwd`, `id`, `whoami` |
| Group | `groupadd`, `groupdel`, `gpasswd`, `groups` |
| Permission | `chmod`, `chown`, `chgrp`, `umask` |
| Proses | `ps`, `top`, `htop`, `kill`, `pkill`, `jobs`, `pgrep` |
| Service | `systemctl start/stop/restart/enable/status` |
| Log | `journalctl`, `tail -f /var/log/syslog` |
