# LAB 02 — User Management

## Tujuan
Peserta mampu membuat dan mengelola user, group, serta memahami file `/etc/passwd`, `/etc/shadow`, dan `/etc/group` di Linux.

## Prasyarat
- Lab 01 selesai
- Akses sudo ke sistem Linux

## Estimasi Waktu
30 menit

---

## Skenario

Anda adalah seorang sysadmin yang baru bergabung di startup teknologi. Anda diminta untuk membuat akun untuk tim baru:

| Nama | Role | Group |
|---|---|---|
| `alice` | Lead Developer | `devteam`, `docker` |
| `bob` | DevOps Engineer | `devteam`, `docker`, `sudo` |
| `carol` | QA Engineer | `devteam` |

---

## Langkah 1: Periksa User yang Ada

```bash
# Tampilkan semua user yang ada
cat /etc/passwd | column -t -s ':'

# Atau menggunakan getent
getent passwd | awk -F: '$3 >= 1000 {print $1, $3, $6}'
```

Perhatikan struktur `/etc/passwd`:
```
username:password:UID:GID:GECOS:home:shell
alice:x:1001:1001:Alice Developer:/home/alice:/bin/bash
```

> `x` di kolom password berarti password disimpan di `/etc/shadow`.

---

## Langkah 2: Buat Group

```bash
# Buat group devteam
sudo groupadd devteam

# Verifikasi group terbuat
grep devteam /etc/group
```

Output yang diharapkan:
```
devteam:x:1002:
```

---

## Langkah 3: Buat User

```bash
# Buat user alice dengan home directory dan bash shell
sudo useradd -m -s /bin/bash -c "Alice Developer" alice

# Set password untuk alice
sudo passwd alice
# Masukkan: Training@123 (atau password pilihan Anda)

# Buat user bob
sudo useradd -m -s /bin/bash -c "Bob DevOps" bob
sudo passwd bob

# Buat user carol
sudo useradd -m -s /bin/bash -c "Carol QA" carol
sudo passwd carol
```

Verifikasi user terbuat:
```bash
id alice
id bob
id carol
```

Output yang diharapkan:
```
uid=1001(alice) gid=1001(alice) groups=1001(alice)
```

---

## Langkah 4: Tambahkan User ke Group

```bash
# Tambahkan alice ke devteam dan docker
sudo usermod -aG devteam alice
sudo usermod -aG docker alice

# Tambahkan bob ke devteam, docker, dan sudo
sudo usermod -aG devteam bob
sudo usermod -aG docker bob
sudo usermod -aG sudo bob

# Tambahkan carol ke devteam
sudo usermod -aG devteam carol
```

> **PENTING:** Gunakan `-aG` (append + group), bukan `-G` saja. Tanpa `-a`, group lain akan terhapus!

Verifikasi keanggotaan group:
```bash
# Cek group alice
id alice
groups alice

# Cek semua anggota devteam
getent group devteam

# Cek isi /etc/group
grep -E "devteam|docker" /etc/group
```

Output yang diharapkan:
```
devteam:x:1002:alice,bob,carol
docker:x:999:alice,bob
```

---

## Langkah 5: Test Login sebagai User Baru

```bash
# Switch ke user alice (harus masukkan password)
su - alice

# Verifikasi identitas
whoami
id
pwd

# Coba buat file di home directory
touch ~/test-alice.txt
ls -la ~/

# Kembali ke user original
exit
```

---

## Langkah 6: Konfigurasi sudo untuk Bob

```bash
# Verifikasi bob ada di grup sudo
groups bob

# Switch ke bob dan test sudo
su - bob
sudo whoami   # Harus output: root

exit
```

---

## Langkah 7: Kunci dan Buka Akun User

```bash
# Lock akun carol (simulasi resign / cuti)
sudo usermod -L carol

# Coba login sebagai carol — akan gagal
su - carol   # Masukkan password carol → harus ditolak

# Buka kembali akun carol
sudo usermod -U carol

# Verifikasi di /etc/shadow (L berarti locked)
sudo grep carol /etc/shadow
```

Perhatikan karakter `!` di depan hash password saat akun dikunci.

---

## Langkah 8: Ubah Informasi User

```bash
# Ubah shell default alice ke zsh (jika zsh terinstall)
# Jika belum install zsh: sudo apt install -y zsh
sudo chsh -s /bin/bash alice

# Ubah GECOS info (nama lengkap)
sudo usermod -c "Alice Lead Developer" alice

# Verifikasi
grep alice /etc/passwd
```

---

## Langkah 9: Buat Script Audit User

Buat script untuk mengaudit semua user di sistem:

```bash
cat > ~/audit-users.sh << 'EOF'
#!/bin/bash
echo "=============================="
echo " LAPORAN USER & GROUP AUDIT"
echo " $(date)"
echo "=============================="
echo ""
echo "=== User dengan UID >= 1000 ==="
getent passwd | awk -F: '$3 >= 1000 && $1 != "nobody" {
    printf "User: %-15s UID: %-6s Home: %s\n", $1, $3, $6
}'

echo ""
echo "=== Anggota Group Devteam ==="
getent group devteam

echo ""
echo "=== User dengan akses sudo ==="
getent group sudo

echo ""
echo "=== User yang dikunci ==="
sudo grep -E ':\!|:\*' /etc/shadow | cut -d: -f1

EOF

chmod +x ~/audit-users.sh
bash ~/audit-users.sh
```

---

## Langkah 10: Cleanup (Hapus User Test)

```bash
# Hapus user beserta home directory-nya
sudo userdel -r carol

# Verifikasi
id carol 2>&1   # Harus output: no such user

# Catatan: alice dan bob tetap ada untuk lab selanjutnya
```

---

## Verifikasi Akhir

Jalankan checklist berikut:

```bash
echo "=== Verifikasi Lab 02 ==="
echo -n "alice ada: "; id alice &>/dev/null && echo "✅" || echo "❌"
echo -n "bob ada: "; id bob &>/dev/null && echo "✅" || echo "❌"
echo -n "alice di devteam: "; groups alice | grep -q devteam && echo "✅" || echo "❌"
echo -n "bob di sudo: "; groups bob | grep -q sudo && echo "✅" || echo "❌"
echo -n "devteam group ada: "; getent group devteam &>/dev/null && echo "✅" || echo "❌"
```

---

## Checkpoint ✅

- [ ] User `alice` dan `bob` berhasil dibuat dengan home directory
- [ ] Group `devteam` terbuat dan memiliki anggota yang tepat
- [ ] `bob` memiliki akses `sudo`
- [ ] Berhasil melakukan `su - alice` dan `su - bob`
- [ ] Script audit berhasil dijalankan
- [ ] Memahami perbedaan `/etc/passwd`, `/etc/shadow`, dan `/etc/group`
