# LAB 03 — File Permission & Ownership

## Tujuan
Peserta mampu mengatur permission dan ownership file/direktori Linux dengan tepat sesuai kebutuhan keamanan dan kolaborasi tim.

## Prasyarat
- Lab 02 selesai (user `alice`, `bob`, group `devteam` sudah ada)
- Akses sudo

## Estimasi Waktu
30 menit

---

## Skenario

Tim `devteam` memiliki proyek web bersama di `/opt/webproject`. Anda perlu mengatur permission agar:
- Semua anggota devteam bisa baca & tulis file proyek
- Hanya owner yang bisa hapus file miliknya
- File script dapat dieksekusi
- Konfigurasi sensitif hanya bisa dibaca oleh owner

---

## Langkah 1: Pahami Permission Saat Ini

```bash
# Buat direktori proyek
sudo mkdir -p /opt/webproject
ls -la /opt/

# Lihat permission dalam format panjang
ls -la /opt/webproject
stat /opt/webproject
```

Output `ls -la`:
```
drwxr-xr-x  2 root root 4096 Jan  1 10:00 webproject
│└──┬──┘└──┬──┘
│   │      └── Others: r-x (baca + masuk, tidak bisa tulis)
│   └───────── Group: r-x
└───────────── Owner: rwx (baca + tulis + masuk)
```

---

## Langkah 2: Set Ownership Direktori Proyek

```bash
# Ubah kepemilikan ke alice (owner) dan devteam (group)
sudo chown alice:devteam /opt/webproject

# Verifikasi
ls -la /opt/ | grep webproject
```

Output yang diharapkan:
```
drwxr-xr-x  2 alice devteam 4096 ... webproject
```

---

## Langkah 3: Set Permission Direktori

```bash
# Set permission: owner rwx, group rwx, others ---
# (Hanya alice dan devteam yang bisa akses)
sudo chmod 770 /opt/webproject

# Verifikasi
ls -la /opt/ | grep webproject
```

Output:
```
drwxrwx---  2 alice devteam 4096 ... webproject
```

---

## Langkah 4: Buat File-File Proyek

```bash
# Switch ke alice dan buat file proyek
su - alice

# Buat file di direktori proyek
cat > /opt/webproject/index.html << 'EOF'
<!DOCTYPE html>
<html>
<body>
  <h1>DevOps Training Project</h1>
</body>
</html>
EOF

cat > /opt/webproject/deploy.sh << 'EOF'
#!/bin/bash
echo "Deploying application..."
cp index.html /var/www/html/ 2>/dev/null || echo "Deploy simulation complete"
EOF

cat > /opt/webproject/.env << 'EOF'
DB_PASSWORD=SuperSecret123
API_KEY=abc123xyz
EOF

# Verifikasi file terbuat
ls -la /opt/webproject/
exit
```

---

## Langkah 5: Atur Permission File

```bash
# Login sebagai alice
su - alice

# File HTML — bisa dibaca semua (644)
chmod 644 /opt/webproject/index.html

# Script deploy — executable oleh owner & group (754)
chmod 754 /opt/webproject/deploy.sh

# File .env — hanya owner yang bisa baca/tulis (600)
chmod 600 /opt/webproject/.env

# Verifikasi semua permission
ls -la /opt/webproject/
exit
```

Output yang diharapkan:
```
-rw-r--r-- 1 alice devteam   85 ... index.html
-rwxr-xr-- 1 alice devteam  102 ... deploy.sh
-rw------- 1 alice devteam   42 ... .env
```

---

## Langkah 6: Test Permission Antar User

```bash
# Test sebagai bob (anggota devteam)
su - bob

# Bisa baca HTML
cat /opt/webproject/index.html    # ✅ Harus berhasil

# Bisa eksekusi deploy.sh
/opt/webproject/deploy.sh         # ✅ Harus berhasil

# Tidak bisa baca .env (permission 600, owner alice)
cat /opt/webproject/.env          # ❌ Harus gagal: Permission denied

# Bisa buat file baru (karena devteam punya wx di direktori)
touch /opt/webproject/bob-notes.txt  # ✅ Harus berhasil

exit
```

---

## Langkah 7: SGID pada Direktori Bersama

Masalah: File baru yang dibuat bob di direktori proyek otomatis dimiliki group `bob`, bukan `devteam`.

```bash
# Cek group dari file yang dibuat bob
ls -la /opt/webproject/bob-notes.txt
# Output: -rw-rw-r-- 1 bob bob ... bob-notes.txt  (group = bob!)
```

**Solusi: Set SGID pada direktori** — file baru akan otomatis mewarisi group direktori.

```bash
# Set SGID
sudo chmod g+s /opt/webproject

# Verifikasi: 's' muncul di posisi execute group
ls -la /opt/ | grep webproject
# Output: drwxrws--- ... alice devteam ... webproject
#                 ^ 's' = SGID aktif

# Test: buat file baru sebagai bob
su - bob
touch /opt/webproject/bob-test2.txt
ls -la /opt/webproject/bob-test2.txt
# Output: ... bob devteam ... (group sekarang devteam!)
exit
```

---

## Langkah 8: Sticky Bit pada Direktori Bersama

Masalah: Anggota devteam bisa **menghapus file milik orang lain** karena mereka punya write permission di direktori.

```bash
# Simulasi: bob menghapus file alice
su - bob
rm /opt/webproject/index.html    # Ini berhasil! (masalah)
exit

# Kembalikan file
su - alice
cat > /opt/webproject/index.html << 'EOF'
<!DOCTYPE html><html><body><h1>DevOps Training</h1></body></html>
EOF
exit

# Pasang Sticky Bit — hanya owner file yang bisa hapus
sudo chmod +t /opt/webproject
ls -la /opt/ | grep webproject
# Output: drwxrws--T ... alice devteam ... webproject
#                    ^ 'T' atau 't' = sticky bit aktif

# Test: bob tidak bisa hapus file milik alice
su - bob
rm /opt/webproject/index.html  # ❌ Harus gagal: Operation not permitted
exit
```

---

## Langkah 9: umask — Default Permission

`umask` menentukan permission default saat file/direktori baru dibuat.

```bash
# Lihat umask saat ini
umask
# Biasanya: 0022

# Cara kerja:
# File baru max: 666 - 022 = 644 (rw-r--r--)
# Direktori baru max: 777 - 022 = 755 (rwxr-xr-x)

# Ubah umask untuk session ini (lebih ketat)
umask 027
touch ~/testfile
mkdir ~/testdir
ls -la ~/ | grep test
# testfile: -rw-r-----  (640)
# testdir:  drwxr-x---  (750)

# Kembalikan ke default
umask 022
```

---

## Langkah 10: Script Audit Permission

```bash
cat > ~/audit-permission.sh << 'EOF'
#!/bin/bash
TARGET="${1:-/opt/webproject}"

echo "================================================"
echo " AUDIT PERMISSION: $TARGET"
echo " $(date)"
echo "================================================"

echo ""
echo "--- Detail File ---"
ls -la "$TARGET"

echo ""
echo "--- File dengan permission berbahaya ---"
# Cari file world-writable
find "$TARGET" -perm -o+w -type f 2>/dev/null | while read f; do
    echo "⚠️  World-writable: $f"
done

# Cari SUID files
find "$TARGET" -perm -4000 -type f 2>/dev/null | while read f; do
    echo "⚠️  SUID: $f"
done

echo ""
echo "--- Ringkasan ---"
find "$TARGET" -type f | while read f; do
    perm=$(stat -c "%a %U:%G %n" "$f")
    echo "$perm"
done

EOF

chmod +x ~/audit-permission.sh
bash ~/audit-permission.sh /opt/webproject
```

---

## Verifikasi Akhir

```bash
echo "=== Verifikasi Lab 03 ==="
echo -n "Direktori proyek ada: "
  [ -d /opt/webproject ] && echo "✅" || echo "❌"
echo -n "Owner alice, group devteam: "
  stat -c "%U:%G" /opt/webproject | grep -q "alice:devteam" && echo "✅" || echo "❌"
echo -n "SGID aktif: "
  [ $(stat -c "%a" /opt/webproject | cut -c1) -eq 2 ] 2>/dev/null || \
  ls -la /opt/ | grep webproject | grep -q 's' && echo "✅" || echo "❌"
echo -n "Sticky bit aktif: "
  ls -la /opt/ | grep webproject | grep -qE 'T|t' && echo "✅" || echo "❌"
echo -n ".env permission 600: "
  [ "$(stat -c %a /opt/webproject/.env 2>/dev/null)" = "600" ] && echo "✅" || echo "❌"
```

---

## Ringkasan Permission Umum DevOps

| Tipe File | Permission | Oktal | Keterangan |
|---|---|---|---|
| Private key SSH | `rw-------` | 600 | Hanya owner bisa baca |
| Config file | `rw-r--r--` | 644 | Owner rw, semua bisa r |
| Shell script | `rwxr-xr-x` | 755 | Semua bisa eksekusi |
| Direktori web | `rwxr-xr-x` | 755 | Standard web dir |
| Direktori tim | `rwxrws---` | 2770 | Hanya tim, SGID aktif |
| Direktori shared | `rwxrwxrwt` | 1777 | /tmp style, sticky bit |

---

## Checkpoint ✅

- [ ] Memahami notasi `rwx` dan nilai oktal
- [ ] Berhasil mengatur ownership dengan `chown`
- [ ] Berhasil mengatur permission dengan `chmod`
- [ ] Membuktikan SGID menyeragamkan group file baru
- [ ] Membuktikan Sticky Bit mencegah penghapusan file orang lain
- [ ] Memahami cara kerja `umask`
