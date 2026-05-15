# LAB 01 — Setup Environment DevOps

## Tujuan
Menyiapkan lingkungan kerja lengkap untuk seluruh rangkaian training DevOps, meliputi instalasi tools dasar, konfigurasi Git, dan verifikasi environment.

## Prasyarat
- Sistem operasi: Ubuntu 22.04 LTS / Debian 12 (atau WSL2 di Windows)
- Akses sudo
- Koneksi internet

## Estimasi Waktu
30 – 45 menit

---

## Langkah 1: Update Sistem

Selalu mulai dengan meng-update daftar paket agar menggunakan versi terbaru.

```bash
sudo apt update && sudo apt upgrade -y
```

Periksa versi OS:

```bash
lsb_release -a
uname -r
```

---

## Langkah 2: Install Tools Dasar

```bash
sudo apt install -y \
  git \
  curl \
  wget \
  vim \
  nano \
  htop \
  net-tools \
  unzip \
  jq \
  tree \
  tmux \
  bash-completion
```

Penjelasan setiap tool:

| Tool | Fungsi |
|---|---|
| `git` | Version control system |
| `curl` | Transfer data via HTTP/HTTPS/FTP |
| `wget` | Download file dari internet |
| `vim` / `nano` | Text editor di terminal |
| `htop` | Monitor proses sistem interaktif |
| `net-tools` | Tools jaringan (ifconfig, netstat) |
| `jq` | JSON processor di command line |
| `tree` | Tampilkan struktur direktori |
| `tmux` | Terminal multiplexer (multi-pane) |

---

## Langkah 3: Install Docker

```bash
# Hapus versi lama jika ada
sudo apt remove -y docker docker-engine docker.io containerd runc 2>/dev/null

# Tambah Docker repository
sudo apt install -y ca-certificates gnupg lsb-release
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Tambahkan user ke grup docker (agar tidak perlu sudo)
sudo usermod -aG docker $USER

# Aktifkan Docker service
sudo systemctl enable docker
sudo systemctl start docker
```

> **Catatan:** Setelah menjalankan `usermod`, logout dan login kembali agar perubahan grup berlaku.

---

## Langkah 4: Konfigurasi Git

```bash
# Set identitas global Git
git config --global user.name "Nama Anda"
git config --global user.email "email@example.com"

# Set editor default
git config --global core.editor vim

# Set branch default
git config --global init.defaultBranch main

# Aktifkan warna output
git config --global color.ui auto

# Tampilkan konfigurasi
git config --list
```

### Generate SSH Key untuk GitHub/GitLab

```bash
# Generate SSH key pair
ssh-keygen -t ed25519 -C "email@example.com" -f ~/.ssh/id_ed25519

# Tampilkan public key (copy ke GitHub/GitLab Settings → SSH Keys)
cat ~/.ssh/id_ed25519.pub

# Start SSH agent dan tambahkan key
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# Test koneksi ke GitHub
ssh -T git@github.com
```

---

## Langkah 5: Install kubectl

```bash
# Download kubectl versi stabil
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Install
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verifikasi
kubectl version --client
```

---

## Langkah 6: Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verifikasi
helm version
```

---

## Langkah 7: Konfigurasi Shell (Opsional tapi Direkomendasikan)

```bash
# Tambahkan alias berguna ke ~/.bashrc
cat >> ~/.bashrc << 'EOF'

# ---- DevOps Aliases ----
alias k='kubectl'
alias d='docker'
alias dc='docker compose'
alias ll='ls -alh'
alias gs='git status'
alias glog='git log --oneline --graph --all'

# Kubectl autocomplete
source <(kubectl completion bash)
complete -F __start_kubectl k
EOF

# Reload shell
source ~/.bashrc
```

---

## Langkah 8: Buat Struktur Direktori Kerja

```bash
mkdir -p ~/devops-labs/{docker,kubernetes,git,cicd,monitoring,security}
tree ~/devops-labs
```

---

## Verifikasi Environment

Jalankan perintah berikut dan pastikan semua menampilkan versi:

```bash
echo "=== Verifikasi Environment ==="
echo -n "OS: "; lsb_release -ds
echo -n "Kernel: "; uname -r
echo -n "Git: "; git --version
echo -n "Docker: "; docker --version
echo -n "Docker Compose: "; docker compose version
echo -n "kubectl: "; kubectl version --client --short 2>/dev/null || kubectl version --client
echo -n "Helm: "; helm version --short
echo -n "curl: "; curl --version | head -1
echo -n "jq: "; jq --version
echo ""
echo "=== Docker Daemon ==="
docker run --rm hello-world
echo ""
echo "✅ Environment siap!"
```

**Output yang diharapkan:**

```
=== Verifikasi Environment ===
OS: Ubuntu 22.04.4 LTS
Kernel: 5.15.0-101-generic
Git: git version 2.34.1
Docker: Docker version 26.x.x, build ...
Docker Compose: Docker Compose version v2.x.x
kubectl: v1.30.x
Helm: v3.x.x
curl: curl 7.81.0 ...
jq: jq-1.6

=== Docker Daemon ===
Hello from Docker!
...

✅ Environment siap!
```

---

## Troubleshooting

### Docker: Permission denied
```bash
# Pastikan user sudah ada di grup docker
groups $USER

# Jika belum, jalankan:
sudo usermod -aG docker $USER
newgrp docker
```

### SSH: Connection refused ke GitHub
```bash
# Coba port 443 sebagai alternatif
ssh -T -p 443 git@ssh.github.com
```

### apt: Repository error
```bash
sudo apt update --fix-missing
sudo dpkg --configure -a
```

---

## Checkpoint ✅

- [ ] Semua package terinstall tanpa error
- [ ] `docker run hello-world` berhasil
- [ ] `git config --list` menampilkan nama dan email
- [ ] SSH key telah ditambahkan ke akun GitHub/GitLab
- [ ] Direktori `~/devops-labs` terbuat
