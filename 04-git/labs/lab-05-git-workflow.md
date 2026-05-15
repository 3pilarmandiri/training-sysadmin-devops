# LAB 05 — Git Workflow

## Tujuan
Mempraktikkan workflow Git lengkap: dari inisialisasi repository, branching, kolaborasi simulasi, hingga menangani merge conflict.

## Prasyarat
- Git sudah terinstall dan terkonfigurasi (Lab 01)
- Akun GitHub (untuk bagian remote)

## Estimasi Waktu
45 menit

---

## Skenario

Anda adalah Lead Developer yang memulai proyek baru "devops-webapp". Anda akan mempraktikkan workflow Git mulai dari awal hingga simulasi kolaborasi tim.

---

## Langkah 1: Inisialisasi Repository

```bash
# Buat direktori proyek
mkdir -p ~/devops-labs/git/devops-webapp
cd ~/devops-labs/git/devops-webapp

# Inisialisasi Git repository
git init

# Verifikasi
ls -la      # Ada direktori .git
git status
```

---

## Langkah 2: Buat Struktur Proyek

```bash
# Buat struktur direktori
mkdir -p src/{controllers,models,views}
mkdir -p tests
mkdir -p docs

# Buat file awal
cat > src/app.py << 'EOF'
#!/usr/bin/env python3
"""DevOps Training Web Application"""

VERSION = "1.0.0"

def main():
    print(f"DevOps WebApp v{VERSION} starting...")
    print("Server running on http://localhost:8080")

if __name__ == "__main__":
    main()
EOF

cat > README.md << 'EOF'
# DevOps WebApp

Proyek latihan untuk training DevOps.

## Cara Menjalankan

```bash
python3 src/app.py
```
EOF

cat > requirements.txt << 'EOF'
flask==3.0.0
pytest==8.0.0
gunicorn==21.2.0
EOF

# Buat .gitignore
cat > .gitignore << 'EOF'
__pycache__/
*.pyc
.env
venv/
*.log
.DS_Store
dist/
.pytest_cache/
EOF
```

---

## Langkah 3: Commit Pertama

```bash
# Cek status — semua file untracked
git status

# Add semua file
git add .

# Cek staging area
git status
git diff --staged

# Commit pertama
git commit -m "feat: initial project structure"

# Verifikasi
git log --oneline
```

Output:
```
d4f8a1b (HEAD -> main) feat: initial project structure
```

---

## Langkah 4: Konfigurasi Branch Utama

```bash
# Buat branch develop dari main
git checkout -b develop

# Verifikasi
git branch
# * develop
#   main
```

---

## Langkah 5: Simulasi Fitur — Branch feature/login

```bash
# Buat branch fitur
git checkout -b feature/login

# Buat file controller login
cat > src/controllers/auth.py << 'EOF'
"""Authentication Controller"""

def login(username: str, password: str) -> dict:
    """
    Proses login user.
    Return: {"success": bool, "token": str, "message": str}
    """
    # Simulasi validasi
    if not username or not password:
        return {"success": False, "token": "", "message": "Username dan password wajib diisi"}
    
    if username == "admin" and password == "admin123":
        return {
            "success": True,
            "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.example",
            "message": "Login berhasil"
        }
    
    return {"success": False, "token": "", "message": "Kredensial tidak valid"}


def logout(token: str) -> dict:
    """Proses logout user."""
    return {"success": True, "message": "Logout berhasil"}
EOF

# Buat unit test
cat > tests/test_auth.py << 'EOF'
"""Unit Test untuk Authentication"""
import sys
sys.path.insert(0, 'src')
from controllers.auth import login, logout

def test_login_success():
    result = login("admin", "admin123")
    assert result["success"] == True
    assert result["token"] != ""

def test_login_empty_credentials():
    result = login("", "")
    assert result["success"] == False
    assert "wajib" in result["message"]

def test_login_invalid():
    result = login("admin", "wrongpassword")
    assert result["success"] == False

def test_logout():
    result = logout("any-token")
    assert result["success"] == True

if __name__ == "__main__":
    print("Running tests...")
    test_login_success()
    test_login_empty_credentials()
    test_login_invalid()
    test_logout()
    print("All tests passed! ✅")
EOF

# Commit perubahan
git add .
git commit -m "feat: tambah auth controller dan unit test"

# Tambah satu commit lagi (simulasi perbaikan)
cat >> src/controllers/auth.py << 'EOF'


def validate_token(token: str) -> bool:
    """Validasi token JWT."""
    return len(token) > 10  # Simplified validation
EOF

git add src/controllers/auth.py
git commit -m "feat: tambah fungsi validate_token"

# Lihat log branch ini
git log --oneline
```

---

## Langkah 6: Merge feature/login ke develop

```bash
# Kembali ke develop
git checkout develop

# Lihat perbedaan sebelum merge
git diff develop..feature/login

# Merge dengan --no-ff agar ada merge commit (best practice)
git merge --no-ff feature/login -m "merge: feature/login ke develop"

# Lihat graph setelah merge
git log --oneline --graph --all
```

Output:
```
*   a1b2c3d (HEAD -> develop) merge: feature/login ke develop
|\
| * f4e5d6c (feature/login) feat: tambah fungsi validate_token
| * g7h8i9j feat: tambah auth controller dan unit test
|/
* d4f8a1b (main) feat: initial project structure
```

```bash
# Hapus branch feature yang sudah selesai
git branch -d feature/login
git branch
```

---

## Langkah 7: Simulasi Merge Conflict

```bash
# Buat dua branch yang mengubah file yang sama

# Branch Alice
git checkout -b feature/alice-update
cat > src/app.py << 'EOF'
#!/usr/bin/env python3
"""DevOps Training Web Application — Updated by Alice"""

VERSION = "1.1.0"
AUTHOR = "Alice"

def main():
    print(f"DevOps WebApp v{VERSION} by {AUTHOR} starting...")
    print("Server running on http://localhost:8080")
    print("Auth module: ENABLED")

if __name__ == "__main__":
    main()
EOF
git add src/app.py
git commit -m "feat: update app.py — Alice menambah auth info"

# Kembali ke develop dan buat branch Bob
git checkout develop

git checkout -b feature/bob-update
cat > src/app.py << 'EOF'
#!/usr/bin/env python3
"""DevOps Training Web Application — Updated by Bob"""

VERSION = "1.1.0"
AUTHOR = "Bob"

def main():
    print(f"DevOps WebApp v{VERSION} by {AUTHOR} starting...")
    print("Server running on http://0.0.0.0:8080")
    print("Environment: PRODUCTION")

if __name__ == "__main__":
    main()
EOF
git add src/app.py
git commit -m "feat: update app.py — Bob ubah config server"

# Kembali ke develop dan merge Alice dulu
git checkout develop
git merge --no-ff feature/alice-update -m "merge: feature/alice-update"

# Coba merge Bob — akan conflict!
git merge --no-ff feature/bob-update
```

Output:
```
Auto-merging src/app.py
CONFLICT (content): Merge conflict in src/app.py
Automatic merge failed; fix conflicts and then commit the result.
```

---

## Langkah 8: Resolve Merge Conflict

```bash
# Lihat file yang conflict
git status

# Lihat isi file conflict
cat src/app.py
```

File akan terlihat seperti:
```python
#!/usr/bin/env python3
<<<<<<< HEAD
"""DevOps Training Web Application — Updated by Alice"""

VERSION = "1.1.0"
AUTHOR = "Alice"

def main():
    print(f"DevOps WebApp v{VERSION} by {AUTHOR} starting...")
    print("Server running on http://localhost:8080")
    print("Auth module: ENABLED")
=======
"""DevOps Training Web Application — Updated by Bob"""

VERSION = "1.1.0"
AUTHOR = "Bob"

def main():
    print(f"DevOps WebApp v{VERSION} by {AUTHOR} starting...")
    print("Server running on http://0.0.0.0:8080")
    print("Environment: PRODUCTION")
>>>>>>> feature/bob-update
```

```bash
# Resolve dengan menggabungkan keduanya
cat > src/app.py << 'EOF'
#!/usr/bin/env python3
"""DevOps Training Web Application"""

VERSION = "1.1.0"
AUTHORS = ["Alice", "Bob"]

def main():
    print(f"DevOps WebApp v{VERSION} starting...")
    print(f"Authors: {', '.join(AUTHORS)}")
    print("Server running on http://0.0.0.0:8080")
    print("Auth module: ENABLED")
    print("Environment: PRODUCTION")

if __name__ == "__main__":
    main()
EOF

# Tandai conflict terselesaikan
git add src/app.py

# Lanjutkan merge
git commit -m "merge: resolve conflict — gabungkan perubahan Alice dan Bob"

# Lihat hasil akhir
git log --oneline --graph --all
```

---

## Langkah 9: Menggunakan Stash

```bash
# Simulasi: sedang mengerjakan sesuatu tapi perlu pindah branch
git checkout develop
echo "# Catatan penting dari rapat" >> docs/meeting-notes.md

# Status: ada file yang belum di-commit
git status

# Tiba-tiba ada hotfix mendesak! Harus pindah branch
# Tanpa stash, checkout akan gagal atau membawa perubahan
git stash save "WIP: catatan rapat"

# Sekarang bisa pindah branch
git checkout main
# ... buat hotfix ...
echo "# Hotfix v1.0.1" >> CHANGELOG.md
git add CHANGELOG.md
git commit -m "hotfix: tambah changelog untuk patch"

# Kembali ke develop
git checkout develop

# Ambil kembali pekerjaan yang distash
git stash list
git stash pop

# Verifikasi
cat docs/meeting-notes.md
git status
```

---

## Langkah 10: Tag Versi Rilis

```bash
# Merge develop ke main untuk simulasi release
git checkout main
git merge --no-ff develop -m "release: merge develop ke main v1.1.0"

# Buat annotated tag
git tag -a v1.1.0 -m "Release v1.1.0 — Tambah fitur authentication"

# Lihat tag
git tag
git show v1.1.0

# Lihat graph lengkap
git log --oneline --graph --all --decorate
```

---

## Langkah 11: Push ke GitHub (Opsional)

```bash
# Buat repository di GitHub terlebih dahulu, lalu:
git remote add origin git@github.com:USERNAME/devops-webapp.git

# Push semua branch
git push -u origin main
git push -u origin develop

# Push semua tag
git push origin --tags

# Verifikasi
git remote -v
git branch -r
```

---

## Langkah 12: Git Alias untuk Produktivitas

```bash
# Tambah alias ke git config
git config --global alias.st "status -s"
git config --global alias.lg "log --oneline --graph --all --decorate"
git config --global alias.co "checkout"
git config --global alias.br "branch"
git config --global alias.undo "reset HEAD~1 --mixed"

# Test alias
git st
git lg
```

---

## Verifikasi Akhir

```bash
cd ~/devops-labs/git/devops-webapp

echo "=== Verifikasi Lab 05 ==="
echo -n "Git repo terinstall: "
  [ -d .git ] && echo "✅" || echo "❌"
echo -n "Branch main ada: "
  git branch | grep -q main && echo "✅" || echo "❌"
echo -n "Branch develop ada: "
  git branch | grep -q develop && echo "✅" || echo "❌"
echo -n "Tag v1.1.0 ada: "
  git tag | grep -q v1.1.0 && echo "✅" || echo "❌"
echo -n "Auth controller ada: "
  [ -f src/controllers/auth.py ] && echo "✅" || echo "❌"
echo -n "Unit test ada: "
  [ -f tests/test_auth.py ] && echo "✅" || echo "❌"

echo ""
echo "=== Git Log ==="
git log --oneline --graph --all --decorate | head -20

echo ""
echo "=== Jalankan Unit Test ==="
python3 tests/test_auth.py
```

---

## Checkpoint ✅

- [ ] Repository berhasil diinisialisasi dengan struktur proyek
- [ ] Memahami staging area dan workflow add → commit
- [ ] Berhasil membuat branch dan merge dengan `--no-ff`
- [ ] Berhasil menangani merge conflict secara manual
- [ ] Berhasil menggunakan stash untuk menyimpan pekerjaan sementara
- [ ] Berhasil membuat annotated tag untuk versi rilis
- [ ] Memahami git log dengan `--graph --all --decorate`
