# BAB 4 — Git & Version Control

## Tujuan Pembelajaran

Setelah menyelesaikan bab ini, peserta mampu:
- Memahami konsep version control dan alasan penggunaannya
- Melakukan operasi Git dasar: init, add, commit, push, pull
- Mengelola branch dan melakukan merge/rebase
- Mengikuti Git workflow yang digunakan di tim DevOps
- Menangani merge conflict

---

## 4.1 Mengapa Version Control?

Bayangkan mengembangkan software tanpa version control:

```
project_v1.zip
project_v1_final.zip
project_v1_final_FINAL.zip
project_v1_final_FINAL_fix.zip
project_v2_alice.zip
project_v2_bob.zip
```

**Masalah nyata:**
- Tidak tahu apa yang berubah antara versi
- Sulit kolaborasi tanpa menimpa pekerjaan orang lain
- Tidak bisa kembali ke versi sebelumnya dengan mudah
- Tidak ada audit trail siapa mengubah apa

**Git menjawab semua masalah ini.**

---

## 4.2 Konsep Dasar Git

### Area Kerja Git

```
Working Directory   Staging Area    Local Repository    Remote Repository
(file di disk)      (index)         (.git/objects)      (GitHub/GitLab)

     ┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐
     │          │      │          │      │          │      │          │
     │ Edit     │─git─▶│ Staged   │─git─▶│ Commit   │─git─▶│  Remote  │
     │ files    │ add  │ changes  │commit│ history  │ push │          │
     │          │      │          │      │          │      │          │
     └──────────┘      └──────────┘      └──────────┘      └──────────┘
          ▲                                    │
          └──────────── git checkout ──────────┘
                        git pull (remote → local)
```

### Tiga State File di Git

| State | Arti |
|---|---|
| **Modified** | File diubah tapi belum di-stage |
| **Staged** | File ditandai untuk commit berikutnya |
| **Committed** | Perubahan tersimpan di database Git lokal |

---

## 4.3 Perintah Git Dasar

### Setup Awal

```bash
# Konfigurasi identitas (wajib sebelum commit pertama)
git config --global user.name "Nama Anda"
git config --global user.email "email@example.com"
git config --global init.defaultBranch main
git config --global core.editor vim
git config --global pull.rebase false   # merge saat pull

# Lihat semua konfigurasi
git config --list
```

### Inisialisasi Repository

```bash
# Buat repository baru
mkdir my-project && cd my-project
git init

# Clone repository yang sudah ada
git clone https://github.com/user/repo.git
git clone git@github.com:user/repo.git         # Via SSH
git clone https://github.com/user/repo.git mydir  # Ke direktori custom
```

### Siklus Kerja Dasar

```bash
# Cek status repository
git status
git status -s    # Format pendek

# Tambah file ke staging area
git add file.txt          # File spesifik
git add src/              # Semua file dalam direktori
git add *.py              # Pattern
git add .                 # SEMUA perubahan
git add -p                # Interaktif (pilih hunk per hunk)

# Buat commit
git commit -m "feat: tambah fitur login"
git commit -am "fix: perbaiki bug validasi"   # Add + commit sekaligus (tracked files)

# Lihat riwayat commit
git log
git log --oneline
git log --oneline --graph --all    # Tampilkan graph semua branch
git log --oneline -10              # 10 commit terakhir
git log --author="Alice"           # Filter by author
git log --since="2 weeks ago"      # Filter by date
git log -- src/api/               # Hanya file tertentu
```

### Melihat Perubahan

```bash
# Diff antara working dir dan staging
git diff

# Diff antara staging dan commit terakhir
git diff --staged

# Diff antara dua commit
git diff abc1234 def5678

# Diff antara branch
git diff main feature/login

# Tampilkan isi commit
git show abc1234
git show HEAD        # Commit terakhir
git show HEAD~1      # Commit sebelumnya
```

---

## 4.4 Branching dan Merging

### Branch Strategy

Branch memungkinkan pengembangan paralel tanpa mengganggu kode stabil.

```
main (production-ready)
│
├── develop (integrasi)
│   ├── feature/login
│   ├── feature/payment
│   └── feature/dashboard
│
└── hotfix/security-patch
```

### Perintah Branch

```bash
# Buat dan pindah branch
git branch feature/login          # Buat branch
git checkout feature/login        # Pindah ke branch
git checkout -b feature/login     # Buat + pindah (shortcut)
git switch -c feature/login       # Cara modern (Git 2.23+)

# List branch
git branch           # Branch lokal
git branch -r        # Branch remote
git branch -a        # Semua branch

# Hapus branch
git branch -d feature/login      # Hapus setelah merge
git branch -D feature/login      # Paksa hapus (belum merge)

# Rename branch
git branch -m old-name new-name

# Push branch ke remote
git push origin feature/login
git push -u origin feature/login   # Set upstream tracking
```

### Merge

```bash
# Merge feature ke main
git checkout main
git merge feature/login

# Jenis merge:
# 1. Fast-forward (tidak ada diverge)
git merge feature/login               # Default

# 2. Merge commit (ada diverge, buat commit baru)
git merge --no-ff feature/login

# 3. Squash merge (gabungkan semua commit jadi satu)
git merge --squash feature/login
git commit -m "feat: tambah fitur login"
```

### Rebase (Alternatif Merge)

```bash
# Rebase feature ke atas main
git checkout feature/login
git rebase main

# Rebase interaktif (edit, squash, reorder commits)
git rebase -i HEAD~3    # Edit 3 commit terakhir

# Setelah rebase, push harus force (karena history berubah)
git push --force-with-lease origin feature/login
```

**Kapan pakai merge vs rebase?**

| | Merge | Rebase |
|---|---|---|
| History | Linear dengan branch | Linear tanpa branch noise |
| Safety | Aman untuk branch publik | Hindari di branch yang dibagi |
| PR/MR | Standar untuk PR | Digunakan untuk cleanup sebelum PR |

---

## 4.5 Menangani Merge Conflict

Conflict terjadi saat dua branch mengubah bagian yang sama pada file.

```bash
# Contoh scenario:
# Alice dan Bob sama-sama mengubah baris 10 di app.js
git merge feature/bob-changes
# AUTO-MERGING app.js
# CONFLICT (content): Merge conflict in app.js
# Automatic merge failed; fix conflicts and then commit.

# Lihat file yang conflict
git status
# both modified: app.js

# Buka file conflict
cat app.js
```

Isi file conflict:
```
function hello() {
<<<<<<< HEAD
    console.log("Hello dari Alice!");
=======
    console.log("Hello dari Bob!");
>>>>>>> feature/bob-changes
}
```

```bash
# Pilih salah satu atau gabungkan secara manual:
# Edit file menjadi:
# function hello() {
#     console.log("Hello dari Alice dan Bob!");
# }

# Setelah resolve, add dan commit
git add app.js
git commit -m "merge: resolve conflict di app.js"

# Atau batalkan merge
git merge --abort
```

---

## 4.6 Remote Repository

```bash
# Tambah remote
git remote add origin https://github.com/user/repo.git
git remote add upstream https://github.com/original/repo.git

# Lihat remote
git remote -v

# Fetch (download tanpa merge)
git fetch origin
git fetch --all

# Pull (fetch + merge)
git pull origin main
git pull                   # Pull dari upstream tracking

# Push
git push origin main
git push origin feature/login
git push --all             # Push semua branch
git push --tags            # Push semua tag

# Delete remote branch
git push origin --delete feature/old-branch
```

---

## 4.7 Git Workflow yang Umum Dipakai

### GitHub Flow (Sederhana)

```
main (production)
  │
  ├── feature/xxx → Pull Request → Code Review → Merge → main
  ├── bugfix/yyy  → Pull Request → Code Review → Merge → main
  └── hotfix/zzz  → Pull Request → Code Review → Merge → main
```

**Aturan:**
1. `main` selalu deployable
2. Buat branch dari `main` untuk setiap fitur/fix
3. Buat Pull Request untuk code review
4. Merge setelah approved

### Gitflow (Lebih Terstruktur)

```
main (production)
  ↑
develop (integrasi)
  ↑
feature/* → merge ke develop
release/* → merge ke develop + main → tag
hotfix/*  → merge ke develop + main → tag
```

---

## 4.8 Perintah Git Lanjutan

### Stash — Simpan Pekerjaan Sementara

```bash
# Simpan pekerjaan yang belum selesai
git stash
git stash save "WIP: sedang refactor auth module"

# List stash
git stash list

# Terapkan stash
git stash apply           # Terapkan stash terbaru (keep stash)
git stash pop             # Terapkan + hapus stash terbaru
git stash apply stash@{2} # Stash tertentu

# Hapus stash
git stash drop stash@{0}
git stash clear           # Hapus semua stash
```

### Tag — Tandai Versi Rilis

```bash
# Buat tag
git tag v1.0.0
git tag -a v1.0.0 -m "Release version 1.0.0"  # Annotated tag

# Push tag
git push origin v1.0.0
git push origin --tags

# List tag
git tag
git tag -l "v1.*"

# Hapus tag
git tag -d v1.0.0-beta
git push origin --delete v1.0.0-beta
```

### Reset dan Revert

```bash
# Batalkan perubahan di working directory
git checkout -- file.txt         # Restore file ke commit terakhir
git restore file.txt             # Git 2.23+ (lebih jelas)

# Unstage file
git reset HEAD file.txt
git restore --staged file.txt    # Git 2.23+

# Reset ke commit tertentu
git reset --soft HEAD~1    # Undo commit, perubahan ke staging
git reset --mixed HEAD~1   # Undo commit, perubahan ke working dir (default)
git reset --hard HEAD~1    # Undo commit + buang semua perubahan (HATI-HATI!)

# Revert (aman untuk branch publik — buat commit baru yang membatalkan)
git revert abc1234
git revert HEAD
```

### Git Bisect — Cari Commit Penyebab Bug

```bash
git bisect start
git bisect bad             # Commit saat ini ada bug
git bisect good v1.0.0     # Versi terakhir yang bagus
# Git otomatis checkout commit di tengah → test → tandai good/bad
git bisect good            # Jika tidak ada bug
git bisect bad             # Jika ada bug
# Ulangi hingga Git menemukan commit penyebab
git bisect reset           # Kembali ke HEAD
```

---

## 4.9 .gitignore

File `.gitignore` menentukan file/direktori yang tidak dilacak Git:

```gitignore
# .gitignore contoh untuk proyek Node.js + Python

# Dependencies
node_modules/
__pycache__/
*.pyc
venv/
.env/

# Build output
dist/
build/
*.egg-info/

# Environment & secrets
.env
.env.local
*.pem
*.key
secrets.yaml

# IDE files
.vscode/
.idea/
*.swp
*.swo

# OS files
.DS_Store
Thumbs.db

# Logs
*.log
logs/

# Test coverage
.coverage
htmlcov/
coverage/
```

---

## Ringkasan Perintah Git

| Kategori | Perintah |
|---|---|
| Setup | `git config`, `git init`, `git clone` |
| Status | `git status`, `git log`, `git diff`, `git show` |
| Perubahan | `git add`, `git commit`, `git restore`, `git reset` |
| Branch | `git branch`, `git checkout`, `git switch`, `git merge`, `git rebase` |
| Remote | `git remote`, `git fetch`, `git pull`, `git push` |
| Lanjutan | `git stash`, `git tag`, `git bisect`, `git cherry-pick` |
